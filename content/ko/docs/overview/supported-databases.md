---
title: 지원하는 데이터베이스
weight: 2
featured: true
---

Vitess는 오픈 소스 SQL DBMS 클러스터를 배포, 확장 및 관리한다. 현재 Vitess는 [MySQL](https://www.mysql.com/), [Percona](https://www.percona.com/software/mysql-database/percona-server) 및 [MariaDB](https://mariadb.org) 데이터베이스를 지원하고 있다.



[VTGate](../../concepts/vtgate/) 프록시 서버는 외부에 자신의 버젼이 MySQL 5.7이라고 응답한다.

## MySQL versions 5.6 ~ 8.0

Vitess는 MySQL 버전 5.6 ~ 8.0의 핵심 기능을 지원하며, 거기엔 [제약사항](../../reference/mysql-compatibility/)이 있다. Vitess는 또한 [Percona Server for MySQL](https://www.percona.com/software/mysql-database/percona-server) 버전 5.6 ~ 8.0을 지원한다.

MySQL 5.6이 2021년 2월 EOL에 들어감에 따라, MySQL 5.7 이상의 버젼의 사용이 권장된다.


## MariaDB versions 10.0 ~ 10.3

Vitess는 MariaDB 버전 10.0 ~ 10.3의 핵심 기능을 지원한다. 10.4 버젼은 [아직 지원하고 있지 않다.](https://github.com/vitessio/vitess/issues/5362)은 MariaDB 버전 10.4를 지원한다.


## 같이 보기

+ [MySQL 호환성](../../reference/mysql-compatibility/)
