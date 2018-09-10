># **_UNDO_RETENTION_**
##### * 작업은 Toad For Oracle Toole을 사용하여 진행을 하였습니다.
##### * 작업을 하게된 이유 및 작업간 이슈사항은 [여기](http://sksggg123.tistory.com/10) 클릭 하시면 확인 하 실수 있습니다.

## UNDO_RETENTION 정의
#### *   일관성 읽기를 위해 제공되는 Undo 데이타의 보유 기간을 결정합니다.
#### *   초기화 파일에서 설정하거나, ALTER SYSTEM 명령을 사용하여 동적으로 수정할 수 있습니다.
#### *   이 parameter는 초 단위로 지정됩니다. 기본값은 900초이며, 이는 Undo 데이타를 15분 동안 보유합니다.
#### *   UNDO_RETENTION을 설정한 후에도 UNDO 테이블스페이스의 크기가 너무 작으면 지정한 시간 동안 Undo 데이타가 보유되지 않습니다.
#### *   UNDO_RETENTION 파라미터는 현재 Undo 테이블스페이스에 UNDO_RETENTION 기간 동안 발생하는 모든 트랜잭션을 수용할 수 있을 만큼 충분한 커야 합니다.
#### *   대용량 이관작업이 있을때 undo tablespace가 비약적으로 상승이 된다.  보통 이럴때는 여분의 datafile를 하는 방법이 가장 쉽지만 datafile이 부족하여 추가가 불가할 경우 undo tablespace를 비워 상승하는 undo를 초기화(?) 시켜주면 임시적으로나마 undo tablespace가 100%에 도달하여 성능적 이슈가 발생하는것을  방지할 수 있다.

## UNDO_RETENTION 절차
1. 임시 undo tablespace 생성
```sql
    create undo tablespce undotbs_temp datafile '~/~.dbf' size 1G;
```
2. 임시로 만든 tablespace 시스템 undo tablespace로 지정
```sql
    alter system set undo_tablespace=undotbs_temp scope=spfile;
        -- spfile or pfile 확인 필요
        select * from v$parameter where name like '%pfile%';
        
        -- undo_retention 임계시간 설정
        --undo tablespace 변경 지정을 하기 위해서는 트랜잭션이 없어야 한다.
        > alter system set "undo_retention" = 0 scope=memory; 
```
3. 기존에 비정상 적으로 확장된 undo tablespace drop
```sql
    -- tablespace에 추가된 datafile 제거 X
    drop tablespace undotbs1;
    -- tablespace에 추가된 datafile 제거 O
    drop tablespace undotbs1 including contents and datafiles;
```
4. 새로 사용할 undo tablespace를 생성
```sql
    create undo tablespce undotbs1 datafile '~/~.dbf' size 1G;
```
5. 새로 만든 undo tablespace를 시스템 undo tablespace로 지정
```sql
    alter system set undo_tablespace=undotbs1;
```
6. 임시로 만든 undotbs_temp drop
```sql
    -- os의 해당 계정으로 접색해서 imsi_undotbs1 테이블스페이스에 속판  파일 rm으로 삭제
    -- 온라인 DB의 경우 접속한 Tool의 세션을 모두 제거하면 *.dbf reuse 가능
    drop tablespace undotbs_temp--including contents and datafiles;
```
7. **_3번_** 의 drop tablespace undotbs1;의 쿼리를 사용했을 경우
```sql
    -- 부족하던 undo tablespace에 붙여둔 datafile를 including contents and datafiles를 하지 않았을때 아래 쿼리로 재사용 가능
    alter tablespace undotbs_temp add datafile '~/~dbf' reuse;
```


#참고 undo retention 확인 쿼리
```sql
    select * from v$parameter where name = 'undo_retention';
```
