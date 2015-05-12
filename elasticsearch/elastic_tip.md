# elastic

* indices 복구하기
 * [Indices Recovery](http://www.elastic.co/guide/en/elasticsearch/reference/current/indices-recovery.html)

## Indices 관리
[elastic : indices APIs](http://www.elastic.co/guide/en/elasticsearch/reference/1.x/indices.html)

### Index 삭제
Index(RDPMS 의 테이블 개념) 의 삭제 방법.
```bash
[root@orion index]# curl -XDELETE 'http://localhost:9200/logstash-2014.12.22'
{"acknowledged":true}
[root@orion index]# 
```


## max_file_descriptors
elasticsearch 의 max_file_descriptors 값 확인
curl http://localhost:9200/_nodes/process?pretty

몇 개의 파일을 open???
* indices 는 각각 지정한 개수의 shards 를 갖는다.
 * indices 가 140개, shards per index 는 5 이므로 700개
* 각 shards 는 지정한 개수의 replicas 를 갖는다.
 * replica 설정 없으므로 pass
* 이것들은(shards) 각각 Lucene index 를 갖는다.
* Lucene index 는 각각 여러 개의 segments 를 갖는다.
 * /elasticsearch-1.4.2/data/sh_cluster1/nodes/0/indices/logstash-2015.05.01/0/index
 * 180 개의 파일이 있음(이것을 segments 라고 하는지는 모르겠지만). 샤드당 180개.
 * 700 * 180 = 126000
* 각 Segment 는 하나의 파일로 구성된다.

ulimit -n 32768 로 했을 때, shards 가 140 가 넘어가는 날 Too many open files 에러가 나면서 elasticsearch 의 장애가 발생.

참고
* [Too many open files warning](http://elasticsearch-users.115913.n3.nabble.com/Too-many-open-files-warning-td4033111.html)

### 장애 : Too many open files

로그를 보면 요런 장애 메시지가 출력. 들어오는 데이터를 저장하지 못하게 된다. elastic 재시작해도 문제는 지속. ulimit -n 명령으로 값을 늘려주어야 한다. 32768 로 설정했는데 이것이 모자라다니... 샤드 140개에 데이터는 120G 정도?
```bash
[2015-04-11 13:18:32,097][WARN ][index.merge.scheduler    ] [ELK Master] [logstash-2015.04.11][1] failed to merge
java.nio.file.FileSystemException: /elasticsearch-1.4.2/data/cluster1/nodes/0/indices/logstash-2015.04.11/1/index/_uy_Lucene410_0.dvm: Too many open files
        at sun.nio.fs.UnixException.translateToIOException(UnixException.java:91)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
        at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
```

### Linux : max open file
리눅스에서 max open file 을 설정하려면.
ulimit -n 으로 입력. 현재의 user 에게만 적용. 최대값은? 구글링하면 65535 를 max 로 설정하는 예제들이 많음. 영구히 적용하려면 /etc/security/limits.conf 를 수정하고 리붓하라고. 토탈 max 값은 sysctl 에서 fs.file-max 값을 변경 적용하라고.

나는 CentOS 6.5
ulimit -n 의 최대값은 1048576 이다. 2의 20승.
리눅스 커널에서 하드코딩되어 있음. (NR_OPEN in /usr/include/linux/fs.h)
```bash
[root@orion index]# sysctl -a |grep max |grep file
fs.file-max = 798319
[root@orion index]# ulimit -n 1048577
-bash: ulimit: open files: cannot modify limit: 명령을 허용하지 않음
```
