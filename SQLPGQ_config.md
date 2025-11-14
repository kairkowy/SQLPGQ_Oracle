## SQL/PGQ 실행을 위한 환경 구성

SQL/PGQ 실행을 위한 소프트웨어 설치 및 환경 구성에 대한 자세한 내용은 오라클 문서를 참고하기 바란다. 여기에서는 주요 작업 내용만 기술할 예정이다.

1. 시스템 환경 요건

- Linux8(7 가능)
- Oracle/Open JDK 11 이상
- Disk space : 2GB
- DB버전 : Oracle23ai 버전 이상 

2. 그래프 패키지 다운로드 

-  Download Oracle Graph Server and Client

oracle-graph-<25.2.0.0.x86_64.rpm : An rpm file to deploy Oracle Graph Server for Linux x86-64.
oracle-graph-client-<ver>.zip : A zip file containing Oracle Graph Client.
oracle-graph-sqlcl-plugin-<ver>.zip : A plugin for SQLcl to run PGQL queries in SQLcl.
oracle-graph-webapps-<ver>.zip : A zip file containing .war files for deploying graph servers in an application server.
oracle-graph-visualization-library-<ver>.zip : A zip file containing a Java Script library for the Graph Visualization application.

3. RPM 패키지 그래프 서버 설치

위에서 다운로드한 파일중에 RPM 파일을 이용하여 스탠드얼론 그래프 서버를 설치한다.1

sodo rpm -i oracle-graph-<version>.rpm

post-installation steps으로 다음 디렉토리 구조 및 파일들이 생성된다.

Creation of a working directory in /opt/oracle/graph/pgx/tmp_data.
Creation of a log directory in /var/log/oracle/graph.
Creation of the oraclegraph operating system group.
Automatic generation of self-signed TLS certificates in /etc/oracle/graph.


User 모드 수정

```
usermod -a -G oraclegraph <graphuser>
```

4. 그래프 셋업 및 준비

DB 접속 및 인증 구성(RPM 설치 경우)

기본 인증 모드

/etc/oracle/graph/server.conf 파일 수정(for Non TLS for case)

```
"enable_tls": true => "enable_tls": false,

```

오라클 지원 툴 사용하는 경우 /opt/oracle/graph/scripts의 quicksetup.sh 에 옵션을 사용하여 수정 가능하다.

```
./quicksetup.sh -d   -- To Disable TLS

```

DB 계정 생성

그래프를 사용할 DB 계정을 생성한다.


그래프 권한 부여

그래프 유저는 그래프 롤을 별도로 부여해줘야 한다. 그래프 롤은 GRAPH_ADMINISTRATOR, GRAPH_DEVELOPER, GRAPH_USER 3가지 유형이 있다. 자세한 내용은 오라클 문서를 참조 바란다.

SYSDBA 권한 유저(SYS or SYSTEM)로 필요한 롤 작업을 우선 실행한다.

```
  DECLARE
    PRAGMA AUTONOMOUS_TRANSACTION;
    role_exists EXCEPTION;
    PRAGMA EXCEPTION_INIT(role_exists, -01921);
    TYPE graph_roles_table IS TABLE OF VARCHAR2(50);
    graph_roles graph_roles_table;
  BEGIN
    graph_roles := graph_roles_table(
      'GRAPH_DEVELOPER',
      'GRAPH_ADMINISTRATOR',
      'GRAPH_USER',
      'PGX_SESSION_CREATE',
      'PGX_SERVER_GET_INFO',
      'PGX_SERVER_MANAGE',
      'PGX_SESSION_READ_MODEL',
      'PGX_SESSION_MODIFY_MODEL',
      'PGX_SESSION_NEW_GRAPH',
      'PGX_SESSION_GET_PUBLISHED_GRAPH',
      'PGX_SESSION_COMPILE_ALGORITHM',
      'PGX_SESSION_ADD_PUBLISHED_GRAPH',
      'PGX_SESSION_SET_IDLE_TIMEOUT');
    FOR elem IN 1 .. graph_roles.count LOOP
      BEGIN
        dbms_output.put_line('create_graph_roles: ' || elem || ': CREATE ROLE ' || graph_roles(elem));
        EXECUTE IMMEDIATE 'CREATE ROLE ' || graph_roles(elem);
      EXCEPTION
        WHEN role_exists THEN
          dbms_output.put_line('create_graph_roles: role already exists. continue');
        WHEN OTHERS THEN
          RAISE;
      END;
    END LOOP;
  EXCEPTION
    when others then
      dbms_output.put_line('create_graph_roles: hit error ');
      raise;
  END;
  /
```

추가 롤 작업

```
  DECLARE
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_CREATE TO GRAPH_ADMINISTRATOR';
    EXECUTE IMMEDIATE 'GRANT PGX_SERVER_GET_INFO TO GRAPH_ADMINISTRATOR';
    EXECUTE IMMEDIATE 'GRANT PGX_SERVER_MANAGE TO GRAPH_ADMINISTRATOR';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_CREATE TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_NEW_GRAPH TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_GET_PUBLISHED_GRAPH TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_MODIFY_MODEL TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_READ_MODEL TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_SET_IDLE_TIMEOUT TO GRAPH_DEVELOPER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_CREATE TO GRAPH_USER';
    EXECUTE IMMEDIATE 'GRANT PGX_SESSION_GET_PUBLISHED_GRAPH TO GRAPH_USER';
    BEGIN
      EXECUTE IMMEDIATE 'GRANT CREATE PROPERTY GRAPH TO GRAPH_DEVELOPER';
    EXCEPTION WHEN others then
      if sqlcode = -990 then
        mdsys.opg_log.debug('grant create property graph to graph_developer: missing privilege, continue');
      else
        raise;
      end if;
    END;
  EXCEPTION
    when others then
      dbms_output.put_line('add_graph_roles_grants: hit error ');
      raise;
  END;
  /
```

RPM 설치한 경우에는 다음 위치에 있는 SQL을 실행하면 간편하게 실행할 수 있다.

```
/opt/oracle/graph/scripts/create_graph_roles.sql

```

grant GRAPH_DEVELOPER, GRAPH_USER, GRAPH_ADMINISTRATOR to graph


DB 접속 구성

RPM으로 설치한 경우 /etc/oracle/graph/pgx.conf 파일을 다음과 같이 수정한다.

pgx_realm 구역에서 jdbc_url 라인 수정

```
pgx_realm": {
  "implementation": "oracle.pg.identity.DatabaseRealm",
  "options": {
    "jdbc_url": "jdbc:oracle:thin:@localhost:1521/freepdb1",
    "token_expiration_seconds": 3600,
```

오라클 지원 툴 사용하는 경우 /opt/oracle/graph/scripts의 quicksetup.sh 에 옵션을 사용하여 수정 가능하다.

```
./quicksetup.sh -u

Enter JDBC URL: jdbc:oracle:thin:@10.0.29.91:1521/freepdb1

```

pgx 서버 실행

```
sudo systemctl daemon-reload pgx

sudo systemctl restart pgx

sudo journalctl -u pgx.service 
```

오라클 그래프 서버 접속 및 실행

브라우저 접속 및 로그인

http://143.47.117.193:7007/dash

id : labadmin, passwd : pwd


PGX에서는 SQL/PGQ 쿼리, PGQL, 인메모리 그래프 쿼리, 일반 오라클 쿼리(DDL, DML) 실행이 가능하다.

----------------------------
end of documents

good luck ~
