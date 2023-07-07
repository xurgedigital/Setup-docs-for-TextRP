# textrp-setup-documentation

This is a step by step guide to deploy textrp on a running kubernetes cluster.
All the yamls are present in this repository in respective folders

Follow below instructions to setup textrp on a new environment:

Prerequisites:
1. A running kubernetes cluster (AWS EKS used here)
2. Ingress controller installed on kubernetes cluster
3. Valid ssl certificate for domain (used *.textrp.io wildcard cert here)

Steps:
1. Setup Postgres service.
	
        a. create persistant volume by applying pv.yaml file
           syntax: kubectl apply -f postgresPv.yaml

        b. create persistant volume claim
	   syntax: kubectl apply -f postgresPVC.yaml
  
        c. create postgres config map to define variables
           syntax: kubectl apply -f pg_configmap.yaml

        d. create postgres deplyment
           syntax: kubectl apply -f postgres_deployment.yaml

        e. expose deployment via service
	   syntax: kubectl expose deployment postgres --type=NodePort --name=postgres
        
        f. test the service:
           - run kubectl get svc

$> kubectl get svc

NAME                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
postgres              NodePort   10.100.119.255   <none>        5432:31266/TCP    11d

           - login to the host and hit the service url: curl http://10.100.119.255:5432

        g. connect to postgres pod and create additional databases for backend, twitter and discord
           syntax: connect to pod and take shell
                   from inside pod:
                   createdb -h localhost -p 5432 -U postgres lucid
                   createdb -h localhost -p 5432 -U postgres  twitter_bridge
		   createdb -h localhost -p 5432 -U postgres  discord_bridge
                   now validate with below command.

root@postgres-f67fd4df9-nnt4h:/# psql -U postgres

psql (13.11 (Debian 13.11-1.pgdg120+1))
Type "help" for help.

postgres=# \l
                               List of databases
      Name      |  Owner   | Encoding | Collate | Ctype |   Access privileges
----------------+----------+----------+---------+-------+-----------------------
 discord_bridge | postgres | UTF8     | C       | C     |
 lucid          | postgres | UTF8     | C       | C     |
 postgres       | postgres | UTF8     | C       | C     |
 synapse        | postgres | UTF8     | C       | C     |
 template0      | postgres | UTF8     | C       | C     | =c/postgres          +
                |          |          |         |       | postgres=CTc/postgres
 template1      | postgres | UTF8     | C       | C     | =c/postgres          +
                |          |          |         |       | postgres=CTc/postgres
 twitter_bridge | postgres | UTF8     | C       | C     |
(7 rows)

postgres=#


2. setup redis cache

       a. apply the deployment file
          syntax: kubectl apply -f redis_deployment.yaml

       b. expose the deployment vi service
          syntax: kubectl expose deployment redis --type=NodePort --name=redis

4. setup synapse service

       a. create persistant volume 
          syntax: kubectl apply -f synapse-pv.yaml

       b. create persistant volume claim
          syntax: kubectl apply -f synapse-pvc.yaml

       c. create initial deployment by using below
          syntax: kubectl apply -f synapse-deployment-initial-setup.yaml

       d. Now delete deployment
          syntax: kubectl get deployments
                  kubectl delete deployment synapse

       e. configure homeserver.yaml
          - login to kubernetes host
   
	  - cd to persistance volume path
            "cd /data/synapse"
     
          - edit homeserver.yaml for the following values:
              server_name: "synapse.textrp.io"
	      public_baseurl: "https://synapse.textrp.io"
              
          - update database section in homeserver.yaml file to look like below:
              database:
  		name: psycopg2
  		args:
		    user: postgres
		    password: postgres
		    database: synapse
		    host: postgres
		    cp_min: 5
		    cp_max: 10
              
           - add oidc provider like below:

              oidc_providers:
  		- idp_id: xumm
		    idp_name: Xumm
		    issuer: "https://oauth2.xumm.app"
		    client_id: "b19848bd-6133-4267-aa72-2bb4a5183893"
		    client_secret: "0ec479fd-5241-4002-b599-99cfc453b6ad"
		    scopes: ["openid","profile"]
		    authorization_endpoint: "https://oauth2.xumm.app/auth"
		    token_endpoint: "https://oauth2.xumm.app/token"
		    userinfo_endpoint: "https://oauth2.xumm.app/userinfo"
		    client_auth_method: client_secret_post
		    skip_verification: true
		    enable_registration: true
		    user_mapping_provider:
		      config:
		        subject_claim: "sub"
		        #localpart_template: "{{ user.account }}"
			#display_name_template: "{{ user.account }}"
 		        #email_template: "{{ user.email }}"
                    
             - edit initial setup deployment file to look like "synapse-deployment.yaml" then apply the deployment file

               syntax: kubectl apply -f synapse-deployment.yaml

             - expose the deployment via service and create ingress to expose via app loadbalancer (url)

               syntax: kubectl expose deployment synapse --type=NodePort --name=synapse
	               kubectl apply -f synapse-ingress.yaml

             - hit the public lb url to test the service
      
             - point the dns "synapse.textrp.io" to public loadbalancer

6. setup element service (app)

    a. apply below kubernetes yamls to bring up element service (synapse url is needed as input)

       syntax: kubectl apply -f elements_configmap.yaml
               kubectl apply -f elements_deployment.yaml
               kubectl expose deployment element-web --type=NodePort --name=element-web-service
               kubectl apply -f elements_ingress.yaml
       
    b.  once lb is launched via ingress service point "app.textrp.io" dns to lb

8. setup synapse-admin service

     a. apply below kubernetes yamls to bring up synapse-admin service (synapse url is needed as input)

        syntax: kubectl apply -f synapse-admin-deployment.yaml
                kubectl expose deployment synapse-admin --type=NodePort --name=synapse-admin
                kubectl apply -f admin-ingress.yaml

     b once lb is launched via ingress service point "admin.textrp.io" dns to lb

10. setup textrp-backend service

     a. Create the following in order following below syntax (two persistance volumes, two persistance volume claims, config map, deployment, expose service and ingress)

        syntax: kubectl apply -f pv-backend.yaml
                kubectl apply -f pv-public.yaml
		kubectl apply -f xlrp-backend-pvc.yaml
		kubectl apply -f xlrp-public-pvc.yaml
		kubectl apply -f backend-configmap.yaml
		kubectl apply -f backend_deployment.yaml
                kubectl expose deployment textrp-backend --type=NodePort --name=textrp-backend
                kubectl apply -f backend-ingress.yaml

     b. once lb is launched via ingress service point "backend.textrp.io" dns to lb

12. setup frontend service

     a. apply below kubernetes yamls to bring up element service (backend url is needed as input)

        syntax: kubectl apply -f frontend-deployment.yaml
                kubectl expose deployment frontend --type=NodePort --name=frontend
                kubectl apply -f frontend-ingress.yaml

     b. once lb is launched via ingress service point "frontend.textrp.io" dns to lb

14. Setup twitter and discord bots

     a. deploy persistance volume, peristance volume claim, deployment yaml and then delete the deployment

        syntax: kubectl apply -f twitter-pv.yaml
                kubectl apply -f twitter-pvc.yaml 
                kubectl apply -f twitter-deployment.yaml
                kubectl expose deployment twitter-bot --type=NodePort --name=twitter-bot
                kubectl delete deployment twitter-bot

     b. now login to kubernetes host and cd to twitter persistant volume

       syntax: cd /data/twitter
               - then edit config.yaml
    
               - update below values: (Note: search for these values and update accordingly, they are not in order)

    		 address: https://synapse.textrp.io
                 appservice:
    			# The address that the homeserver can use to connect to this appservice.
    			address: http://10.100.31.201:29327                   --------------------> This is twitter-bot service ip and port

                 permissions:
        		'*': user
        		'@admin:synapse.textrp.io': admin
 
                 database: postgres://postgres:postgres@postgres:5432/twitter_bridge?sslmode=disable

     c. now again run deployment with "kubectl apply -f twitter-deployment.yaml"

        new registration yaml will be generated in persistant volume
        copy that newly generated registration.yaml to synapse persistance volume with name "mautrix-twitter-registration.yaml"
     
     d. now update synapse homeserver.yaml file with below values

        app_service_config_files:
 	 #- /data/mautrix-discord-registration.yaml
  	 - /data/mautrix-twitter-registration.yaml 
        
         Note: uncomment discord value once you have generated doscord registration file and copied to synapse persistance volume

     e. restart synapse pod for new homeserver config to take effect

        restart twitter-bot pod for bot to connect to synapse

     f. follow the exact same steps for enabling discord bot.

16. Now the you can test the application.
