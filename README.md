# Debugging Docker

## Commands to run container
To run the docker container:
- I completed the challenge on a Xenial64 Vagrant box and configured the box to forward ports guest:8080, and host:8080
- Once you've made the changes in the Vagrant box, reload the box `vagrant reload`
- Clone the repo `git clone https://github.com/nickolasteixeira/Dubugging_Docker.git`
- Install docker if not already installed via the install script `./install_docker.sh` 
- Then run the docker compse file `sudo docker-compose up`
- Wait for the messages to stop running before pinging `localhost:8080` on your host browser

Below were my thoughts and steps for solving this problem.

## DEBUGGING PROCESS
1. I decided to work in a Vagrant box, so I create an ubuntun/xenial64 box and wrote a file name `install_docker.sh` that would install docker ce and docker-compose
2. Once the installation was complete, I tried to run the `docker-compose up -d` and ran into a few issues.
- I kept getting permission denied issues and docker deamon socket issues and quickly realized I need to use the `sudo` command
3. The next issue I ran into was trying to map my Vagrant box to show the changes to the host machine on localhost:8080.
- I had to configure the Vagrant box to map the forward port between the guest Vagrant box and the host machine
- config.vm.network "forwarded_port", guest: 8080, host: 8080
4. Once I made the changes, I was running into issues docker issues about multiple programs trying to use port 80.
- This part took me a while because I didn't have much docker-compose.yml experience. After reading the docker file, searching the web and asking peers for help, I realized that multiple applications were trying to use port 80 and have to change the ports nginx was maping the host:container. Rabbit was currently using port 80, so I decided to flip the ports and change the -ports code to "8080:80"
- Error Message: `ERROR: for ingestion  Cannot start service ingestion: driver failed programming external connectivity on endpoint insight_ingestion_1 (8d95245958d2733db4a3d55a559b8b84d36571fcdf80e06de007d5893ef2854c): Bind for 0.0.0.0:80 failed: port is already allocated`
- FIX: `sudo docker-compose up -d --build`
5. After fixing the compose file, I then tried to run the application again. Now I could check the host machine on localhost:8080 and at least see some output.
6. There was html output, but there were no numbers associated with the success rate.
- I decided to check the logs for each container
- I found this command to be really useful: sudo docker logs --timestamps [CONATINER_NAME]
- I started to look into the logs files of each container and realized that postgres was having an issue finding the db name, users and password. I then had to export those environment variables to my machine
- Once exported, I then realized that the postgres commands were trying to run, but no 'weblogs' databas was set up. I then entered into the web_db container and created a 'weblogs' database and fixed the issue.
7. Once the database was set up, I rechecked each log file of each container
- The NGINX container was working correctly with 200 status codes
- The Flask container was working correctly
- The web ingestion container was not working correctly, so I then tried to debug that container
8. I spent a good amount of time trying to debug the web_ingestion container.
- Once I found out that the issues was in the ingestion.py file under the ingestion folder, I tried to debug that file and thought that it could potentially be an import error.
- I then added a __init__.py file to the ingestion folder, `sudo docker restart [container]` and tried to access the application again, but wasn't able to fix the problem.
- I tried to use different import methods and restart the container, but wasn't successful.
- In the ingestion folder, I ran python3.7 to get to the interpreter so I could try the import statement. It worked correctly. Now I know it's a container issue.
- I initially thought it was an issue with the container, so I kept trying to restart the container and try to rebuild all the containers, `sudo docker-compose up -d --build`
- I tried to use a few commands like: `sudo docker-compose` down, then `sudo docker-compose up -d`. No luck.
- This wasn't the issue. After spending a few hours on learning how to restart, rebuild and create containers, I still wasn't able to fix the problem.
- I tried to use `sudo docker exec -it [web_log_container], but wasn't able to access the container because the container wasn't being built and was trying to restart. It kept trying to restart on repeat and didn't allow me to enter into the container because that container was never getting built because of the import error.
- Then I decided to just build a seperate container with the web_log_container image. My thought process was to create the container, enter into it, then try to create a python3.7 interpreter to try and reproduce the `from utils import module` error. Once I realized it didn't work, I then discovered the utils.py file wasn't in the container.
- Once I realized that, I went to the Dockerfile. I then found the issue. The utils.py module wasn't being copied to the container. I then added the line of code that made sure all the files ending in .py get added to the container. I then added a .dockerignore file to not add .pyc files
9. I then rebuilt the container: `sudo docker-compose up -d --build` and checked the application again.
- SUCCESS! I was now recieving the correct output.
10. I then rechecked all the other container log files and realized one other small error regarding pyscopg2-binaries, so I entered into the container for Postgres and pip installed the dependecy.


## TESTS:

11. Once I manually completed the task, I then had to test it by cloning the repository and into another directory and running `sudo docker-compose up -d`. To my suprise, there were errors. The goal of this challenge was to find the problem and automate the solution.
- Instead of manually entering the postgres container and adding the table, I had to do it in the `app.py` file.
12. After adding the table, I starting to find other errors.
```
2018-10-26T19:19:47.276707192Z     conn = psycopg2.connect(host='db', database=os.environ['POSTGRES_DB'], user=os.environ['POSTGRES_USER'], password=os.environ['POSTGRES_PASSWORD'])
2018-10-26T19:19:47.276752434Z   File "/usr/local/lib/python3.7/site-packages/psycopg2/__init__.py", line 130, in connect
2018-10-26T19:19:47.276960115Z     conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
2018-10-26T19:19:47.277137838Z psycopg2.OperationalError: could not translate host name "db" to address: Temporary failure in name resolution`
```
- This error didn't show up previously when runing the application. What was happening was that flask and ingest was depending on the `db` application, but flask and ingest was trying to find the `db` application before it was fully set up. I had to create a `wait_db.sh` bash script that waiting for the `db` application to complete before executing the rest of the dependences. 

13. Lastly, I was running into error when refreshing the page because I had the `app.py` file contain the code to create the `weblogs` table. It instead should have been fixed in the `docker-entrypoint-initdb.d/init-tables.sh` file. I tried to fix the `init-tables.sh`, but found out that I could instead create an `.sql` file that gets executed with the proper sql syntax to initilize the POSTGRES database.
- Problem solved.

14. I realized instead of exporting the environments onto your vagrant box, you could simply add them in the docker-compose file at the root of the directory. That way any machine you try to run the docker command on will work.

## BONUS:
15. This part wasn't too difficult, minus one thing. I realized in the ingestion.py file RabbitMQ was doing a json dump with just the day and the status values, so I added the source value to the dictionary. 
- Once I added that, I needed to change something in the processing.py file. This was the tricky part. I saw the `values` variable in the callback function had the date and status, but no source, so I added the source. For some reason, I had to add backticks to denote it as a string, and not a variable.
- Once the processing was set up, I had to add a column in the database.
- Then on the applicaiton side, I had to wrtie a few more SQL queries to grab objects that only had remote or local and then pass them as objects to the front end.
- Presto! The front-end now displays total 200 requests, local and remote requests.
 
