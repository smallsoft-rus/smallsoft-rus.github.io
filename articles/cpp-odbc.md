# Подключение к базам данных в С++ с помощью ODBC

Хотя С++ обычно используется для низкоуровневых программ, иногда в нем возникает необходимость работать с реляционными базами данных. Для этого существует стандартный программный интерфейс ODBC. В данной статье мы рассмотрим использование ODBC на примере Windows, SQL Server Express и Visual Studio.

## Обзор ODBC

[Open Database Connectivity (ODBC)](https://docs.microsoft.com/en-us/sql/odbc/microsoft-open-database-connectivity-odbc) - процедурный программный интерфейс, который позволяет прикладным программам выполнять SQL-запросы к различным источникам данных и получать результаты в табличной форме. Модуль, который добавляет поддержку определенных источников данных (т.е. СУБД), называется драйвером. ODBC можно использовать в С/С++ напрямую, в .NET через управляемые обертки, или в других языках, если их разработчики предоставляют для этого стандартные библиотеки.

ODBC реализует несколько стандартов: 

- The Open Group CAE Specification [Data Management: SQL Call-Level Interface (CLI)](https://pubs.opengroup.org/onlinepubs/009654899/toc.pdf)
- [ISO/IEC 9075-3:1995](https://www.iso.org/standard/24357.html) - Call-Level Interface (SQL/CLI)
- [ISO/IEC 9075:1992](https://www.iso.org/standard/16663.html) - Database Language SQL (SQL-92)

В Windows SDK для Visual C++ реализация ODBC находится в заголовочном файле sqlext.h и библиотеках odbc32.lib и odbccp32.lib.

Основные функции ODBC, которые нам понадобятся для подключения к БД:

- `SQLDriverConnect` - Подключение к источнику данных
- `SQLPrepare` - Подготовка запроса (компилирует запрос, чтобы последующие многократные выполнения этого запроса происходили быстрее)
- `SQLExecute` - Выполнение запроса
- `SQLFetch` - Получение записи из результатов запроса
- `SQLGetData` - Получение поля записи
- `SQLDisconnect` - Закрытие соединения с источником данных

Список всех функций можно посмотреть в [ODBC Reference](https://docs.microsoft.com/en-us/sql/odbc/reference/syntax/odbc-reference).

## Источники данных ODBC

Для подключения к источнику данных необходимо, чтобы на компьютере был установлен драйвер. В Windows входят стандартные драйверы для следующих источников:

- Microsoft Access (2003 и ранее)
- Microsoft Excel (2003 и ранее)
- Microsoft SQL Server (старая версия)
- Paradox
- dBASE
- Текстовые файлы

Все эти драйвера 32-битные. Так как DLL драйвера грузится приложением, это значит, что их можно использовать только из 32-битного приложения. 64-битные драйвера включены только в серверные Windows, и для их использования нужен MDAC 2.7 SDK ([ODBC 64-Bit Information](https://docs.microsoft.com/en-us/sql/odbc/reference/odbc-64-bit-information)). 

Чтобы использовать другие СУБД или добавить поддержку 64-битных приложений в клиентских ОС, понадобится скачать и установить драйвера. Например, драйвер SQL Server можно скачать здесь: https://docs.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server

Источник данных задается строкой подключения, которая имеет следующий вид:

    Driver={<Driver>};DSN='';SERVER=<Server>;DATABASE=<Database>;

**Параметры строки соединения**
    
*Driver:* имя драйвера. Распространенные значения: 

- `SQL Server`
- `SQL Server Native Client XX.0` (где ХХ - версия. Например SQL Server Native Client 11.0 для SQL 2012)
- `Microsoft Access Driver (*.mdb, *.accdb)`
- `Microsoft Excel Driver (*.xls)`

Драйвер "SQL Server" - это старый стандартный драйвер, который включен в Windows, но не поддерживает возможности новых версий. SQL Server Native Client - это уже более новая версия, которую надо устанавливать.

- *Server:* путь к экземпляру вида `(IP или имя сервера)\(имя экземпляра)`. Точка означает localhost. Если используется экземпляр по умолчанию, имя экземпляра и косую черту нужно опустить
- *Database:* имя БД, уже присоединенной к серверу SQL Server.
- *AttachDBFileName:* путь к файлу БД SQL Server для подключения (Внимание: При таком сценарии он будет открыт монопольно, пока вы работаете с соединением!).
- *DBQ:* путь к файлу БД для Access и других СУБД
- *CREATE_DB:* путь к файлу БД, если его нужно создать
- *DSN:* имя именованного источника данных (его можно создать в панели управления: **Администрирование** - **Источники данных ODBC**)

Полный список доступен в документации: [DSN and Connection String Keywords and Attributes](https://docs.microsoft.com/en-us/sql/connect/odbc/dsn-connection-string-attribute)

Примеры строк соединения для разных драйверов:

    Driver={SQL Server};DSN='';SERVER=.\sqlexpress;DATABASE=mydatabase;
    Driver={Microsoft Excel Driver (*.xls)};DSN='';CREATE_DB="C:\test\newfile.xls";DBQ=C:\test\newfile.xls;READONLY=0;
    Driver={Microsoft Access Driver (*.mdb, *.accdb)};DSN='';DBQ=C:\users.mdb

## Пример кода для вывода таблицы из БД

Следующий пример кода на С++ демонстрирует подключение к БД и вывод в консоль содержимого таблицы. Он предполагает, что у вас локально развернут SQL Server `.\sqlexpress` с базой данных base, в которой создана таблица Users.

```с++
#include <stdio.h>
#include <Windows.h>
#include <sqlext.h>
#include <locale.h>

WCHAR szDSN[] = L"Driver={SQL Server};DSN='';SERVER=.\\sqlexpress;DATABASE=base;";
WCHAR query[] = L"SELECT * FROM Users";

void DisplayError(SQLSMALLINT t,SQLHSTMT h) {

    //Получение информации об ошибке SQL
    SQLWCHAR       SqlState[6], Msg[SQL_MAX_MESSAGE_LENGTH];
    SQLINTEGER    NativeError;
    SQLSMALLINT   i, MsgLen;
    SQLRETURN     rc;

    SQLLEN numRecs = 0;
    SQLGetDiagField(t, h, 0, SQL_DIAG_NUMBER, &numRecs, 0, 0);  

    i = 1;
    while (i <= numRecs && (rc = SQLGetDiagRec(t, h, i, SqlState, &NativeError,
            Msg, sizeof(Msg), &MsgLen)) != SQL_NO_DATA) {
        wprintf(L"Error %d: %s\n", NativeError, Msg);
        i++;
    }
}

int main()
{   
    HENV    hEnv = NULL;
    HDBC    hDbc = NULL;
    HSTMT hStmt = NULL;
    int iConnStrLength2Ptr;
    WCHAR szConnStrOut[256];
    SQLINTEGER rowCount = 0;
    SQLSMALLINT fieldCount = 0, currentField = 0;
    SQLWCHAR buf[128],colName[128]; 
    SQLINTEGER ret;

    RETCODE rc; //Код статуса ODBC API

    setlocale(LC_ALL, "Russian");

    /* Выделение дескриптора среды */
    rc = SQLAllocEnv(&hEnv);
    /* Выделение дескриптора соединения */
    rc = SQLAllocConnect(hEnv, &hDbc);

    /* Подключение к БД */
    rc = SQLDriverConnect(hDbc, NULL, (WCHAR*)szDSN,
        SQL_NTS, (WCHAR*)szConnStrOut,
        255, (SQLSMALLINT*)&iConnStrLength2Ptr, SQL_DRIVER_NOPROMPT);

    if (SQL_SUCCEEDED(rc))
    {
        /* Подготовка запроса SQL */
        rc = SQLAllocStmt(hDbc, &hStmt);
        rc = SQLPrepare(hStmt, (SQLWCHAR*)query, SQL_NTS);      

        /* Выполнение запроса */
        rc = SQLExecute(hStmt);
        if (SQL_SUCCEEDED(rc))
        {
            wprintf(L"\n- Columns -\n");

            SQLNumResultCols(hStmt, &fieldCount);
            if (fieldCount > 0)
            {   
                for (currentField = 1; currentField <= fieldCount; currentField++)
                {
                    SQLDescribeCol(hStmt, currentField,
                        colName, sizeof(colName), 0, 0, 0, 0, 0);
                    wprintf(L"%d: %s\n", (int)currentField, colName);
                }
                wprintf(L"\n");

                /* Получение записей из результатов запроса */                               

                rc = SQLFetch(hStmt);
                while (SQL_SUCCEEDED(rc))
                {
                    wprintf(L"- Record #%d -\n", (int)rowCount);

                    for (currentField = 1; currentField <= fieldCount; currentField++)
                    {
                        rc = SQLGetData(hStmt, currentField, SQL_C_WCHAR, buf, sizeof(buf), &ret);

                        if (SQL_SUCCEEDED(rc) == FALSE) {
                            wprintf(L"%d: SQLGetData failed\n", (int)currentField);
                            continue;
                        }

                        if (ret <= 0) {
                            wprintf(L"%d: (no data)\n", (int)currentField);
                            continue;
                        }

                        wprintf(L"%d: %s\n", (int)currentField, buf);
                    }                   

                    wprintf(L"\n");
                    rc = SQLFetch(hStmt);
                    rowCount++;
                };                  

                rc = SQLFreeStmt(hStmt, SQL_DROP);

            }
            else
            {
                wprintf(L"Error: Number of fields in the result set is 0.\n");
            }                   

        }
        else {
            wprintf(L"SQL Failed\n");
            DisplayError(SQL_HANDLE_STMT, hStmt);
        }
    }
    else
    {
        wprintf(L"Couldn't connect to %s\n", szDSN);    
        DisplayError(SQL_HANDLE_DBC, hDbc);
    }

    /* Отключение соединения и очистка дескрипторов */
    SQLDisconnect(hDbc);
    SQLFreeHandle(SQL_HANDLE_DBC, hDbc);
    SQLFreeHandle(SQL_HANDLE_ENV, hEnv);

    getchar();
    return 0;
}
```
