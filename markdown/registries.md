#Registry Differences

|Key| T3AIGOWEB01|Q3BIGOWEB01|
|-----------------|----------|---------------------|
|APPSVR|A|B *(Different as of 6/2)*|
|APPSVR_INSTALLED|Y|N *(Different as of 6/2)*|
|DB_LOGIN_NAME|*Empty String*|cossnet|
|DB_NAME|*Empty String*|cossnet|
|DBSVR|A|B *(Different as of 6/2)*|
|DBSVR_INSTALLED|Y|N *(Different as of 6/2)*|
|DNS_NAME|aaa-uat1.ipipeline.com<br>*(domain did not reply to ping)*|*Empty String*|
|SMTP_SVR_ADDR|*Empty String*|mta.ipipeline.us <br>*(domain did not reply to ping)*|
|USE_DNS_NAME|B|A|
|WEBSVR_ADDR|aaa-uat1.ipipeline.com<br>*(domain did not reply to ping)*|10.128.145.75 *(this machine)*|

### Note: I pinged aaa-uat1.ipipeline.com, and the DNS resolved to 216.21.246.187, however the ping timed out. 
