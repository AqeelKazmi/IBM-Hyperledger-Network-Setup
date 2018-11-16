# IBM-Hyperledger-Network-Setup
This repository provides the way to setup hyperledger network on ibm cloud.

1. Place your fabric codes folder inside "contracts" folder, then create bna using the following command.
  
  **composer archive create -t dir -n contracts/hyperledger-power-watson -a contracts/dist/hyperledger-power-watson.bna**

2. Then create Blockchain service https://console.bluemix.net/catalog/services/blockchain
   2a. ## Starter Plan
   2b. ## Enterprise Plan

   For both plans set your versions by following the [link given](https://console.bluemix.net/docs/services/blockchain/develop_install.html#installing-a-development-environment)
   Once you have set your local environment according to selected plan, proceed with further commands.

3. In the Network Monitor of Blockchain service, Change to the 'Overview' page and press the Connection Profile button and      in the popup press Download. Rename your file to connection-profile.json and move it to the contracts/dist folder in your    cloned copy of this repository. Open your connection-profile.json file and scroll to the bottom. In the registrar field      there is an enrollSecret property. Copy it and use in the following command to generate CA card.

    **composer card create -f ca.card -p contracts/dist/connection-profile.json -u admin -s <ENROLL_SECRET>**

4. Import CA card:

   **composer card import -f ca.card -c ca**
 
5. To generate Certificates:
   
   **composer identity request --card ca --path ./credentials -u admin -s <ENROLL_SECRET>**

  In Blockchain service, go to 'Member' page. Click on certificates. Popup will be opened.Above command will generate a       credentials directory. Copy the contents of credentials/admin-pub.pem to your clipboard and paste in the  certificate       textbox of the popup. Submit the popup. Restart the peers. Finally sync the certificates to the channel by opening the       'Channels' page and in the selected Channel press the three dots in the actions column to open the menu. Click Sync         Certificate and then Submit in the popup.

6. Create a card with channel and peer admin roles:

   **composer card create -f adminCard.card -p ./contracts/dist/connection-profile.json -u admin -c ./credentials/admin-pub.pem -k ./credentials/admin-priv.pem --role PeerAdmin --role ChannelAdmin**

7. Import the admin card:

  **composer card import -f adminCard.card -c adminCard**

8. Install the network:
  
  **composer network install -c adminCard -a ./contracts/dist/hyperledger-power-watson.bna**

9. Start the network:
  
  **composer network start -c adminCard -n hyperledger-power-watson -V 0.2.5 -A admin -C ./credentials/admin-pub.pem -f delete_me.card**

10. Ping the network:
  
   **composer network ping -c admin@hyperledger-power-watson**

   **_If successfuly ping, it will display something like this:
    The connection to the network was successfully tested: hyperledger-power-watson
        Business network version: 0.2.5
        Composer runtime version: 0.19.18
        participant: org.hyperledger.composer.system.NetworkAdmin#admin
        identity: org.hyperledger.composer.system.Identity#04d0b64e144a1734765a9ed00a84c07a36bc8faa2e3e593ea136f5912509bf7f
_**

11. Create [Cloudant service](https://console.bluemix.net/catalog/services/cloudantNoSQLDB). In the Cloudant dashboard use       the left navigation bar to go to the 'Service credentials page'. Press New credential then press Add in the popup           leaving the name as Credentials-1.

12. Create a new file called cloudant.json in your working root directory and paste the following JSON into it:
     **{
      "composer": {
         "wallet": {
             "type": "composer-wallet-cloudant",
             "options": {}
         }
     } 
   }**

   Get the JSON data of your credentials by clicking View credentials on the Credentials-1 row of the 'Service credentials      page'. Replace the data in the options field of your cloudant.json file with this JSON adding an additional field to the    copied JSON with the value "database": "wallet"

13. Create the Cloudant database using the value in the JSON for the url field:

    **curl -X PUT <CLOUDANT_URL>/wallet**

14. Set the NODE_CONFIG environment variable on your machine using the contents of your cloudant.json file with new lines       removed:

    **export NODE_CONFIG=$(awk -v RS= '{$1=$1}1' < cloudant.json)**

15. Import the admin card to the Cloudant service:
   
    **composer card import -f ./admin@hyperledger-power-watson.card**

16. Deploy the Application:
    **cf login**
    **set api endpoints: cf api https://api.eu-gb.bluemix.net** 
    **Enter email and password**

17. Push the REST server using the docker image:

     **cf push hyperledger-power-rest --docker-image hyperledger/composer-rest-server:0.20.3 -i 1 -m 256M --no-start --no-manifest --random-route**

    **_Note: Use rest server version as per your selected plan (Starter plan or Enterprise Plan)_**

18. Set the NODE_CONFIG environment variable for the REST server:
    
    **cf set-env hyperledger-power-rest NODE_CONFIG "$NODE_CONFIG"**

19. Set the other environment variables for the REST server:

    **cf set-env hyperledger-power-rest COMPOSER_CARD admin@hyperledger-power-watson**
    **cf set-env hyperledger-power-rest COMPOSER_NAMESPACES required**
    **cf set-env hyperledger-power-rest COMPOSER_WEBSOCKETS true**

20. Push playground using the docker image:

    **cf push hyperledger-power-playground --docker-image hyperledger/composer-playground:0.20.3 -i 1 -m 256M --no-start --random-route --no-manifest**

   **_Note: Use playground version as per your selected plan (Starter plan or Enterprise Plan)_**

21. Set the NODE_CONFIG environment variable for the playground:

    **cf set-env hyperledger-power-playground NODE_CONFIG "$NODE_CONFIG"**

22. Generate Angular Application using Yeoman. Push the application using the files in your clone of this repository:

    **cf push hyperledger-power-watson-app -f ./apps/hyperledger-power-watson/manifest.yml -i 1 -m 128M --random-route --no-start**

23. Bind blockchain service to the application.
    **cf bind-service hyperledger-power-watson-app <BLOCKCHAIN_SERVICE_NAME> -c '{"permissions":"read-only"}'**


24.  Set the environment variable used to tell the application where to send requests. Go to rest-server, click on routes,        select edit routes. Copy the both urls (Separated by dot):

     **cf set-env hyperledger-power-watson-app REST_SERVER_URLS '{"hyperledger-power-rest": {"httpURL": "https://<REST_SERVER_URL>/api", "webSocketURL": "wss://<REST_SERVER_URL>"}}'**

25. Set the environment variable used to tell the application where the playground is located.  Go to playground server,         click on routes, select edit routes. Copy the both urls (Separated by dot):
   
   **cf set-env hyperledger-power-watson-app PLAYGROUND_URL 'https://<PLAYGROUND_URL>'**

26. #Start the application
 
     Start the REST server:
       **cf start vehicle-manufacture-rest**

    Start the playground:
      **cf start vehicle-manufacture-playground**

    Start the vehicle manufacture application:
      **cf start vehicle-manufacture**



Helpful Links:
https://console.bluemix.net/docs/services/blockchain/develop_install.html#installing-a-development-environment
https://hyperledger.github.io/composer/v0.19/installing/update-dev-env
https://github.com/IBM-Blockchain/vehicle-manufacture
https://console.bluemix.net/docs/services/blockchain/develop_enterprise.html#deploying-a-business-network
