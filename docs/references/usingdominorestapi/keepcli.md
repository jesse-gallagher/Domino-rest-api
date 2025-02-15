# Domino REST API CLI

## Overview

A simple shell script using the curl command is all you need to interact with Domino REST API.

Create the following shell script <!--(macOS, Linux, Windows version below)--> in your `~/bin` directory and enable it using `chmod +x Domino REST API`.

<!--I like to-->Skip the `.sh` extension to make calling the command similar to Windows `.cmd` files.

### For Linux or macOS

```bash
#!/bin/bash
# Simple keep command line
#Global variables
httpcommand=$(echo "$1" | tr '[:lower:]' '[:upper:]')
keep=https://frascati.projectkeep.io
user="Doctor Notes"
apipath=api

# Login function
execute_login() {
   # Login to KEEP remote
   read -p "server [$keep]:" server
   read -p "Username [$user]: " uservar
   read -sp "Password: " passvar
   server=${server:-$keep}
   uservar=${uservar:-$user}
   echo
   echo accessing $server as $uservar
   echo
   dataraw="{\"username\" : \"${uservar}\", \"password\" : \"${passvar}\"}"
   token=$(curl --location --request POST "${server}/api/v1/auth" \
   --header 'Content-Type: application/json' \
   --data-raw "${dataraw}" |  jq -r  '.bearer')
   echo $token
   echo token=$token > $TMPDIR/keep.login
   echo server=$server >> $TMPDIR/keep.login
   exit 0
}

# Logout, just remove the token
execute_logout() {
    echo User logged out > $TMPDIR/keep.login
    rm -f $TMPDIR/keep.login
    echo User logged out
    exit 0
}

# Make a call to keeep
execute_http() {
if [ ! -f $TMPDIR/keep.login ]; then
	echo "You need to login first with keep login"
	exit 1
fi
export $(cat $TMPDIR/keep.login | xargs)
 httpcommand=$(echo "$1" | tr '[:lower:]' '[:upper:]')
 shift
 curl -s -X ${httpcommand} --fail --insecure -H "Authorization: Bearer ${token}" -H "Content-Type: application/json" ${server}/${apipath}/$@
 if [ $? -ne 0 ]; then
   echo "Command $httpcommand failed!"
   exit 1
 fi
 exit 0
}

execute_help() {
  echo "Usage keep [login|logout|help]"
  echo "      keep [get|post|put|patch|delete] URL PAYLOAD"
  echo " e.g. keep login"
  echo "      keep get v1/scopes"
  echo "      keep get pim-v1/inbox"
  echo "Hints:"
  echo '      "keep get" without parameters - returns list of APIs'
  echo "      common APIs: v1,admin-v1,pim-v1,poi-v1,setup-v1"
  echo '      "keep get [api]/schema/[schema-json-fiel from keep get]" returns the OpenAPI spec for that API'
  echo " e.g. keep get v1/schema/openapi.basis.json"
  echo '      using "| jq" after a command returns pretty printed JSON (jq installed separately)'
}

if [ -z "$1" ]; then
  execute_help;
  exit 1
fi

if [[ "$httpcommand" == "LOGIN" ]]; then
  execute_login;
fi

if [[ "$httpcommand" == "LOGOUT" ]]; then
  execute_logout;
fi

if [[ "$httpcommand" == "HELP" ]]; then
  execute_help;
  exit 0
fi

# All other cases
execute_http $@

```

### For Windows

```bash
@ECHO OFF
REM Poor man's keep command line
SETLOCAL EnableDelayedExpansion

SET K_tempfile=%TEMP%\keep.login

WHERE /Q curl
IF %ERRORLEVEL% NEQ 0 GOTO needcurl
WHERE /Q powershell
IF %ERRORLEVEL% NEQ 0 GOTO needps

SET "K_ps=powershell -Command "\"%1\".ToLower()""
FOR /F "usebackq delims=" %%p IN (`%K_ps%`) DO SET K_httpcommand=%%p

SET K_keep=http://localhost:8880
SET K_user=KEEP Admin
SET K_apipath=api

IF "%K_httpcommand%"=="help" GOTO execute_help
IF "%K_httpcommand%"=="login" GOTO execute_login
IF "%K_httpcommand%"=="logout" GOTO execute_logout
GOTO execute_http

:execute_login
SETLOCAL EnableDelayedExpansion
IF EXIST %K_tempfile% DEL /Q %K_tempfile%
SET /P K_server=server [%K_keep%]: %=%
IF "%K_server%"=="" SET K_server=%K_keep%
SET /P K_uservar=Username [%K_user%]: %=% 
IF "%K_uservar%"=="" SET K_uservar=%K_user%
SET "K_ps2=powershell -Command "$pword = read-host 'Password' -AsSecureString ; ^
    $BSTR=[System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($pword); ^
        [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)""
FOR /F "usebackq delims=" %%p IN (`%K_ps2%`) DO SET K_passvar=%%p
ECHO accessing %K_server% as %K_uservar%
ECHO.

SET K_dataraw={\"username\":\"%K_uservar%\",\"password\":\"%K_passvar%\"}

REM ECHO Performing 'curl --location --request POST %K_server%/api/v1/auth --header "Content-Type: application/json" --data-raw "%K_dataraw%"'
FOR /F "delims=" %%i in ('curl --silent --location --request POST %K_server%/api/v1/auth --header "Content-Type: application/json" --data-raw "%K_dataraw%"') DO SET K_curlout=%%i

FOR /F "tokens=1,2 delims=:{}, " %%A IN ("%K_curlout%") DO (
    IF "%%~A"=="bearer" SET K_token=%%~B
)
IF "%K_token%"=="" GOTO loginfailed
ECHO %K_token%
ECHO.
ECHO Login successful
ECHO.
ECHO token=%K_token%>%TEMP%\keep.login
ECHO server=%K_server%>>%TEMP%\keep.login
GOTO end

:execute_logout
ECHO User logged out > %K_tempfile%
IF EXIST %K_tempfile% DEL /Q %K_tempfile%
ECHO.
ECHO User logged out
ECHO.
GOTO end

:execute_http
IF NOT EXIST %K_tempfile% GOTO needtologin
FOR /F "tokens=1,2 delims==" %%a IN (%K_tempfile%) DO (
  IF %%a==token SET K_token=%%b
  IF %%a==server SET K_server=%%b
)

SET "K_ps3=powershell -Command "\"%K_httpcommand%\".ToUpper()""
FOR /F "usebackq delims=" %%p IN (`%K_ps3%`) DO SET K_httpcommand=%%p

SET K_curloutfile=%TEMP%\keep.curlout
IF EXIST %K_curloutfile% DEL /Q %K_curloutfile%

FOR /F "tokens=1,*" %%a in ("%*") do set K_allbutfirstparm=%%b
REM ECHO Performing `curl --silent -X %K_httpcommand% --fail --show-error --insecure -H "Authorization: Bearer %K_token%" -H "Content-Type: application/json" %K_server%/%K_apipath%/%K_allbutfirstparm%'

curl --silent -X %K_httpcommand% --fail --show-error --insecure -H "Authorization: Bearer %K_token%" -H "Content-Type: application/json" %K_server%/%K_apipath%/%K_allbutfirstparm% >> %K_curloutfile%
IF NOT EXIST %K_curloutfile% goto error_executing
TYPE %K_curloutfile%
ECHO.
IF EXIST %K_curloutfile% DEL /Q %K_curloutfile%
GOTO end

:execute_help
ECHO.
ECHO Usage keep [login^|logout^|help]
ECHO       keep [get^|post^|put^|patch^|delete] URL PAYLOAD
ECHO e.g.  keep login
ECHO       keep get v1/scopes
ECHO       keep get pim-v1/inbox
ECHO Hints:
ECHO       "keep get" without parameters - returns list of APIs"
ECHO       common APIs: v1,admin-v1,pim-v1,poi-v1,setup-v1
ECHO       "keep get [api]/schema/[schema-json-fiel from keep get]" returns the OpenAPI spec for that API
ECHO  e.g. keep get v1/schema/openapi.basis.json
ECHO       to use a body in a request #1, use --data-binary "@pathToYourJsonFile"
ECHO       to use a body in a request #2, use --data-raw and enclose your data in double quotes
ECHO            for Windows, you must escape all commas (^^,) and quotes (\") in the json
ECHO            example: --data-raw "{\"Form\":\"Customer\"^,\"First_Name\":\"Joe\"^,\"last_name\":\"Notesguy\"}"
ECHO       using "| jq" after a command returns pretty printed JSON (jq installed separately)
ECHO.
GOTO end

:needcurl
ECHO.
ECHO Cannot find curl on your path.  
ECHO Please install curl before trying to use this tool.
ECHO.
GOTO end

:needps
ECHO.
ECHO Cannot find PowerShell on your path.  
ECHO Please install PowerShell before trying to use this tool.
ECHO.
GOTO end

:needtologin
ECHO.
ECHO You need to login first with keep login.
ECHO.
GOTO end

:loginfailed
ECHO.
ECHO Failed to log in.
ECHO.
GOTO end

:error_executing
ECHO.
ECHO Your command failed to execute
ECHO.
GOTO end

:end
SET K_tempfile=
SET K_ps=
SET K_httpcommand=
SET K_keep=
SET K_user=
SET K_apipath=
SET K_server=
SET K_uservar=
SET K_ps2=
SET K_passvar=
SET K_dataraw=
SET K_curlout=
SET K_token=
SET K_ps3=

```

## Using the CLI

You need to log in using the call `keep login`. Use "keep HTTPKEYWORD urlpath" for example `keep get databases` or `keep pim get inbox` to access the API. For the http keywords `POST`, `PUT` and `PATCH`, you must provide the `Content-Type` header and a body. See `man curl` for details.
