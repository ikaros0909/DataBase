# DataBase
Database 관련 정보


## [삭제 된 데이터 복원]  
http://www.sqler.com/bColumn/873012  
테이블에서 데이터가 삭제된 것을 복원하는 방법에 대해서 알아 본다. 실습용 데이터베이스와 테이블을 생성한다.  
전제조건 : Full Backup이 존재해야함. 복구모델이 전체 이어야 함.  

### Create DB.
```sql
USE	[master];  
GO
CREATE	DATABASE	ReadingDBLog;
GO
```

### Full backup  
```sql
backup database ReadingDBLog to disk = 'C:\SQL_Backup\ReadingDBLog.bak'  
```
  
### Create tables.  
```sql
USE	ReadingDBLog;
GO
CREATE	TABLE	[Location]	(
[Sr.No]	INT	IDENTITY,
[Date]	DATETIME	DEFAULT	GETDATE	(),
[City]	CHAR (25) DEFAULT	'Seoul');
```

- 생성된 테이블에 데이터를 입력 한다.  
```sql
USE	ReadingDBLog
go
INSERT	INTO	Location	DEFAULT	VALUES	;
GO 100
```

- 일부 데이터를 삭제 한다. 10 이하의 행이 삭제 된 것을 확인 할 수 있다.  
```sql
USE	ReadingDBLog
Go
DELETE	Location
WHERE	[Sr.No]	< 10
go

select * from Location
```


트랜잭션 로그에서 삭제 된 행에 대한 정보를 얻기 위해 다음의 스크립트를 실행 한다.   
Delete 구문이 실행된 트랜잭션 ID를 확인 할 수 있으며 AllocUnitName 열에서 테이블의 이름을 확인 할 수 있다.   
트랜잭션ID가 동일하게 사용된 것은 삭제 된 행이 일괄적으로 수행 되었다는 것을 뜻한다.  
  
```sql
use	ReadingDBLog
go

SELECT
    [Current LSN],
    [Transaction ID],
    Operation,
    Context,
    AllocUnitName
FROM
fn_dblog(NULL,	NULL)
WHERE	Operation	=	'LOP_DELETE_ROWS'
```


다음 명령은 트랜잭션ID를 사용하여 LOP_BEGIN_XACT 작업의 LSN을 확인 할 수 있다.     
또한 작업이 시작된 시간을 확인 할 수 있다.  

```sql
USE	ReadingDBLog
go
SELECT
    [Current LSN],
    Operation,
    [Transaction ID],
    [Begin Time],
    [Transaction Name],
    [Transaction SID]
FROM
fn_dblog(NULL,	NULL)
WHERE	[Transaction ID]	=	'0000:00000407'
    AND	[Operation]	=	'LOP_BEGIN_XACT'
```


다음은 데이터를 복구하기 위해 16진수의 LSN 값을 변경하는 것이다.   
데이터 복구를 위해 STOPBEFORREMARK 작업을 사용한다.   
STOPBEFOREMARK 작업을 실행하려면 16진수 값을 사용한다.  
LSN의 값 00000022:00000107:0001에서 [:] 부호를 기준으로 3개의 파트로 나누어서 변환 한다.  
Hex to Decimal Convert 변환기 : http://www.binaryhexconverter.com/hex-to-decimal-converter  
34:290:1  
A파트의 10진수는 그대로 사용하고 B파트의 경우 10자리의 자릿수를 사용. C파트의 경우 5자릿수를 사용한다.   
따라서 다음과 같이 LSN 조합이 생성된다.  
34000000026300010  
  
  
마지막으로 트랜잭션 백업을 실행하고 새로운 서버에 전체 백업 및 로그 백업을 복원한다.  

```sql
backup log ReadingDBLog	to disk = 'C:\SQL_Backup\ReadingDBLog.trn'
```
  
  
SQL Server 운영 체제 오류 5 : "5 (액세스가 거부되었습니다.)   
제어판 ->시스템 및 보안 ->관리 도구 ->서비스 ->SQL Server (SQLEXPRESS) 두 번 클릭 -> 마우스 오른쪽 단추로, 속성   
로그온 탭 선택 "로컬 시스템 계정"을 선택하십시오 (기본값은 다소 둔한 Windows 시스템 계정이었습니다)   
-> OK 오른쪽 클릭, 중지 오른쪽 클릭, 시작   

```sql
RESTORE	DATABASE ReadingDBLog_COPY FROM	DISK = 'C:\SQL_Backup\ReadingDBLog.bak'
WITH	REPLACE,	NORECOVERY,
MOVE	'ReadingDBlog'	TO	'C:\SQL_Backup\ReadingDBLog.mdf',
MOVE	'ReadingDBlog_log'	TO	'C:\SQL_Backup\ReadingDBLog.ldf'
GO
```

--Restore Log backup with STOPBEFOREMARK option to recover exact LSN.
```sql
RESTORE	LOG	ReadingDBLog_COPY FROM DISK	= N'C:\SQL_Backup\ReadingDBLog_1.trn'
WITH NORECOVERY

RESTORE	LOG	ReadingDBLog_COPY FROM DISK	= N'C:\SQL_Backup\ReadingDBLog_2.trn'
WITH STOPBEFOREMARK	= 'lsn:34000000026300010', RECOVERY
```

### 특정지점으로 복원하기  
먼저 전체백업 데이터베이스를 Restore 한 다음 trn 파일을 이용해 특정 시점으로 복원한다.
``` 특정시점 복원
RESTORE	LOG	ReadingDBLog_COPY FROM DISK	= N'D:\DBBackup\DSS2020_0928_19.trn'
WITH STOPAT = '2017-11-11 03:03:03', RECOVERY
```

복원이 완료되면 다음과 같이 데이터를 조회한다. 삭제된 데이터가 트랜잭션로그 백업에서 복구 된 것을 확인 할 수 있다.  

```sql
USE	ReadingDBLog_COPY
GO
SELECT * from Location
```
### DB가 복원중일때에는 다음과 같이 온라인으로 변경
```sql
RESTORE DATABASE LAIGODB WITH RECOVERY
```


### 트랜잭션 로그 축소
```dbcc
/*DB축소*/
--ALTER DATABASE DSS2022 SET RECOVERY SIMPLE   
--DBCC SHRINKdatabase (DSS2022,truncateonly)  
--DBCC SHRINKFILE(DSS2022_log)  
--ALTER DATABASE DSS2022 SET RECOVERY FULL   

/*DB정보*/  
--EXEC sp_helpdb AILES2013  

/*DB통계업데이트*/
exec sp_updatestats
```
### 특점 시점 복원 예시
```
Restore FileListOnly FROM DISK = '/var/opt/mssql/data/DSS2021_1026.bak'
GO

RESTORE	DATABASE DSS2022_20211026 FROM	DISK = '/var/opt/mssql/data/DSS2021_1026.bak'
WITH	REPLACE,	NORECOVERY,
MOVE	'DSS2022'	TO	'/var/opt/mssql/data/DSS2022_20211026.mdf',
MOVE	'DSS2022_log'	TO	'/var/opt/mssql/data/DSS2022_20211026_log.ldf'
GO

RESTORE LOG DSS2022_20211026 FROM DISK = N'/var/opt/mssql/data/DSS2022_20211026.trn' WITH STOPAT = '2021-10-26 09:00:00', RECOVERY
GO

RESTORE DATABASE DSS2022_20211026 WITH RECOVERY
GO
```

### 이기종 누적백업
1) TRN이 “진짜 PAMS 체인”인지 확인
-- TRN 파일의 메타정보 확인: DatabaseName, DatabaseGUID, BackupType, FirstLSN/LastLSN 등
```
RESTORE HEADERONLY
FROM DISK = N'D:\DBRoot\pams\pams_20250828.trn';
```
여기서 확인할 포인트:
BackupType = 2(Log) 이어야 함
DatabaseName이 PAMS여도 DatabaseGUID가 다르면 같은 DB로 취급되지 않음

FirstLSN/LastLSN이 직전에 복원한 FULL의 checkpoint_lsn 이후를 연속으로 덮어야 함
(FULL의 기준값은 아래 2)에서 확인)

2) 복원한 FULL의 기준 LSN/체인 확인
-- FULL 백업 헤더 (체인 기준 LSN/DatabaseGUID 확인)
```
RESTORE HEADERONLY
FROM DISK = N'D:\DBRoot\pams\PAMS_20230531.bak';
```
확인 포인트:
DatabaseGUID: TRN과 일치해야 함
BackupType = 1(Full)
CheckpointLSN, DatabaseBackupLSN 값을 메모
원리: DatabaseGUID가 다르면 그 TRN은 이 FULL 위에 절대 적용 불가.
이름(PAMS)이 같아도 GUID가 다르면 다른 체인이야.

3) TRN 파일에 백업 세트가 여러 개일 수 있음 → FILE=n 지정
하나의 .trn 파일 안에 여러 번의 로그 백업이 들어 있을 수 있어. 이때는 **정확한 세트 번호(FILE=n)**를 골라야 한다.

-- 파일 안의 모든 백업 세트 나열(간단 요약 보려면 msdb 쿼리도 가능)
```
RESTORE HEADERONLY
FROM DISK = N'D:\DBRoot\pams\pams_20250828.trn';
```
-- 위 결과에서 원하는 세트의 Position 값을 FILE=n에 넣어 실행
```
RESTORE LOG [PAMS]
  FROM DISK = N'D:\DBRoot\pams\pams_20250828.trn'
  WITH FILE = <Position값>, NORECOVERY;
```
