# jmeter-test-task
- [Instruction](#instruction)
- [Test Plan structure](#test-plan-structure)
- [Issues for api under test](#issues-for-api-under-test)

## Instruction
1. Install JMeter (https://jmeter.apache.org/download_jmeter.cgi)
2. Add *%jmeter_folder%/bin* to PATH
3. Clone the repository, open the repo folder
4. Launch run.cmd  
Result: there will be created the following folder structure:
    - .\reports
    - .\reports\%timestamp% - contains output.csv with test results
    - .\reports\%timestamp%\dashboard - contains html-report (index.html)

## Test Plan structure
- User Defined Variables:  
    - *BASE_USERNAME* - HTTP basic authentication username  
    - *BASE_PASSWORD* - HTTP basic authentication password

- HTTP Request Defaults:  
    Contains server name and HTTP Request path for all requests

- Thread Group  
    Number of Threads (users) : *${__P(threads)}* - can be passed as cmd-argument  
    Loop Count:                 *${__P(loops)}*   - can be passed as cmd-argument  

    - Random Variable:  
        *PREFIX* - need for generating usernames and email addresses
    
    - Get a guest token (simple controller)
        - Encode uname + pwd string (BeanShell PreProcessor)  
            Need for encoding *BASE_USERNAME* and *BASE_PASSWORD* to Base64 and passing  
            them to Authentification Header as *${base64HeaderValue}* variable
        - HTTP Header Manager  
            Need for basic authorization with *${base64HeaderValue}* encoded username : password
        - POST Get a guest token  
            */v2/oauth2/token*  
            - Extract access token (JSON Extractor)  
                Need to extract access_token from the response and store it to the *ACCESS_TOKEN* variable. It will be reused for user registration request
            - Assert response code is 200 (Response Assertion)  
  
    - Register a new player (simple controller)
        - Encode user password (BeanShell PreProcessor)  
            Need for encoding *BASE_PASSWORD* to base64 string (for authorization header)
        - HTTP Header Manager  
            Need for Authorization Bearer with *${ACCESS_TOKEN}*
        - POST Register a new player  
            */v2/players*  
            - Body Data:  
                There is used *${PREFIX} + ${__time()}* combination to avoid duplicates
            - Assert response code is 201
            - JSON Extractors: 
                - Extract USER_ID
                - Extract USER_NAME
                - Extract EMAIL
            - Set properties (BeanShell Assertion) 
                Need to set extracted variables to properties (to make them visible for all threads).  
                Actually it doesn't requiered for the particular tests, but I decided to keep them just in case of future needs.

    - Player authorization (simple controller)
        - HTTP Header Manager  
            Need for basic authorization with ${base64HeaderValue} encoded username : password
        - POST Player authorization  
            Log in the player with *${USER_NAME}* from the previous request (POST Register a new player)
            - Assert response code is 200
            - Extract ACCESS_TOKEN (JSON Extractor)  
                *ACCESS_TOKEN* will be reused for all subsequent requests
            - Set property ACCESS_TOKEN (BeanShell Assertion)  
                Make *ACCESS_TOKEN* a property to enable visibility for all threads

    - Transaction controller  
        Just to organize child requests
        - HTTP Header Manager  
            Contains Authorization Bearer with *${__property(ACCESS_TOKEN)}* for all requests

        - Get all players (Loop Controller)  
            Loops Count: 10  
            - GET Get all players  
                */v2/players*  
                Get request to fetch information about all created users
                - Assert response code is 200
            - GET Get a single player  
                Get request to fetch information about user with id *${USER_ID}* (extracted from the user registration response)
                - JSR223 Assertion  
                    Verify the response structure and response code
                    
        - Get games list (loop controller)  
            Loops Count: 10  
            - GET Get games  
                */v2/games*  
                Get request to fetch information about all created games  
            - Assert response code is OK  
        - Get games list with sorting (loop controller)  
            Loops Count: 10  
            - GET Get games  
                */v2/games?sort=name*  
                Get request to fetch sorted list with all created games  
            - Assert response code is OK  

## Issues for api under test
1. Register a new player *POST /v2/players HTTP/1.1* can be performed without *currency_code* parameter  
    - **Expected result**: *currency_code* parameter can be sent: string, required, default currency code according to ISO4217
    - **Actual result: error** "Invalid currency_code". Player can be created without *currency_code* parameter
2. Get games list *GET /v2/games/* returns an **empty list**  
    - **Expected result**: *GET /v2/games* retrieving a games collction
    - **Actual result**: response contains an **empty list**  
    The same issue for *GET /v2/games?sort=name* - the empty list is retrieved.     
    The request *GET /v2/games/${GAME_ID}* returns an error - there is no games created.   
3. Get all players *GET /v2/players* retrieves information only for the **last created user**.
    - **Expected result**: *GET /v2/players* retrieves a users list
    - **Actual result**: response contains only one user (the last created). Some of the tests fail due to this error.
    - Error example: *{"name":"Not Found","message":"Object not found: 6041","code":0,"status":404}*
    