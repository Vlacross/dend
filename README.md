This repo was created to hold a guide on how to configure a custom bash program on an ec2 that allows one to add endpoints to an apache server using docker.

This process has been built off of my <a href="https://github.com/Vlacross/dauto">dauto project</a>.

Previously, I gathered resources I used to walk through setting up a simple ec2 instance hosting a static site from an apache server running on a docker container. The intent was to use this process as a hands on exercise as well as to get my hands on a little docker and AWS documentation.

The actual use of the project as a development server led me to realize that every time I made a change to the site I was hosting, I would push it to github, ssh into my ec2 instance, stop my apache container, scrap it, rebuild it so that it could grab a fresh version of the repo, and run it.
While the process isn't terribly complicated, it was complex enough that I wrote all the steps into a bash program to make the rebuild process easier.

<h2 style="text-decoration: underline;">dend</h2>

Next, I wanted to be able to host multiple sites and apps using environment variables without needing to rebuild my base apache container every time as this was becoming time consuming. 

So for the sake of saving time, I used my pre-existing <a href="https://github.com/Vlacross/RSVP">message-board app</a> that uses a Mongo database via an express server.
All I had to do was add a <a href="https://github.com/Vlacross/RSVP/blob/master/Dockerfile">simple Dockerfile.</a>
You can pull the repo to your ec2 home:

        git pull https://github.com/Vlacross/RSVP.git


For the Mongo Database, Docker made it simple with their ready-to-go mongo image:

        docker pull mongo

<a href="src: https://github.com/dockerfile/mongodb/issues/32">src</a>

And their docs for creating a <a href="https://docs.docker.com/v17.09/engine/userguide/networking/#user-defined-networks">user-defined-network</a> and <a href="https://docs.docker.com/network/network-tutorial-standalone/">configuring it</a> made things move smoothly when it came to connecting containers.

After pulling the mongo image, build an image with:

        docker build -t mongo

Now before running it, let's make our network bridge:

        docker network create --driver bridge mdb_hub

Now we can start our mongo container and configure it on our new network:

        docker run --rm -d --net=mdb_hub --name mongo mongo

--rm   removes the container after it's stopped

-d     starts the container in the background


Now because my message-board app was made for a slightly different network configuration, our database uri is different.
Running this inside the app folder we pulled from github should work:



        sed -i "3, 4c const MONGODB_URI = process.env.NODE_ENV === \'development\' ? \'mongodb://mongo/RSVP\' : process.env.MONGODB_URI; \nconst MONGODB_URI_TEST = process.env.NODE_ENV === \'development\' ? \'mongodb://mongo/RSVP-test\' : process.env.MONGODB_URI_TEST;" config.js


and if for whatever reason it doesn't, we simply need to replace the lines in the config file that are directing our uri variable to localhost(third and fourth lines), with the following:

        const MONGODB_URI = process.env.NODE_ENV === 'development' ? 'mongodb://mongo/RSVP' : process.env.MONGODB_URI;

        const MONGODB_URI_TEST = process.env.NODE_ENV === 'development' ? 'mongodb:/mongo/RSVP-test' : process.env.MONGODB_URI_TEST; 


Once that is changed, make sure we are in the app folder and run:

        docker build -t nodock .

which should build an image for our app which we can run via:

        docker run -d --rm --net=mdb_hub --name nodock -p 8080:8080 nodock


Specifying a name makes it easier to handle the container later.

run :

        docker container ls

to make sure both containers are running. If your node container starts seemingly fine and shuts down after 30 seconds or so, double check the mongo database uri.

If the container hangs, you can remove it with:

        docker rm nodock --force

Remember to rebuild the container after making changes to any of the files.

If everything goes according to plan and both our mongo container and app server stay up, we can test our connection by seeding some data from our app:

        docker exec -t nodock bash -c 'node utils/seed-database.js'

This should seed some mock data into our database which we can check by logging into our mongo container with:

        docker exec -ti mongo bash

which should drop us into a tty shell, where we can run:

        show dbs

We should see RSVP amidst some other built in databases (admin, config, local)

If not, double check the database uri and ports to make sure everything is configured properly.

At this point, we should have an app running and interacting with a database in a user-defined network. We are now ready to add a Virtual Host to our Apache server to forward traffic to to our containerized app.

Make a folder named <strong>dauto_endpoint</strong> in your ec2 home directory and pull this repo:

        mkdir dauto_endpoint && cd dauto_endpoint && \
        git pull https://github.com/Vlacross/dend.git

then run:

      chmod +x dend

which should make the script executable.

Test this with:

        ./dend --help

you should see a help message.

You can add this directory to your ec2 PATH via:

        sed -i "$(cat .bashrc | wc -l)a \export PATH="/home/\$USER/dauto_endpoint:$PATH"" .bashrc

then source your rc file:

        source /home/$USER/.bashrc

test this with:

        dend -h

If you see the help message again, everything worked as expected.

Now since we have our app running, if you used the commands given in these instructions, it should be named <strong>nodock</strong>(if you used a different name in this guide, use it in place of <strong>nodock</strong>).

Since are going to be forwarding traffic between containers, we need to move our apache container to our new network:

        docker stop <apache-container-name-or-id>
        
        docker run --rm -ti -d --net=mdb_hub -p 80:80 --name <apache-image> <apache-container-name>


To add an endpoint to your ec2 site enter:

        dend --add

which will prompt you to enter a name for your new endpoint. For example purposed, lets say we enter <strong>newpoint</strong> 

Then it will ask you to enter the name of the container you want to host (in my case, I entered nodock).

Supposing the network configuration worked as planned, you should see the terminal spit out the IP and Port of the container and show that apache enabled the endpoint and reloaded.

To test you can visit your ec2 in a browser with your chosen endpoint.

<strong>Make sure to add a forward slash after the endpoint, as Apache VirtualHosts can be finicky about address syntax.</strong>


        http://ec2<hyphen-seperated-IPv4>.<ec2-region>.compute.amazonaws.com/<new endpoint>/

Here we have a functioning app in a live environment to test and experiment with! 
<strong>Delete any endpoints when you stop a container, or if a container crashes </strong>

To delete an endpoint:

        dend --delete

You will be prompted for the endpoint name. if you can't remember the spelling, use:

        dend --list

to show current endpoints.


<h4 style="text-decoration: underline;">Edgecase - troubleshoot</h4>
When you add an endpoint, dend creates a file in the dauto_endpoint folder. It uses these to recognize current endpoints. If you delete one on accident, you can either make the file yourself, or manually disable the endpoint and add it again.

To disable the endpoint, log into your apache container:

        docker exec -ti <apacher-container-name> bash

You'll drop into a tty. Look for your endpoint-config file:

        ls -al /etc/apache2/sites-available

and disable it:

        a2dissite <endpoint-name>

then reload the apache service:

        service apache2 reload

now it is safe to remove the config file:

        rm /etc/apache2/sites-available/<endpoint-name>.conf

then simply exit out of the container and add it again.


If a container stops and another starts before the first one is started again, they may switch IPs which would mean the created endpoint would lead to the second container.
Some unresolved catches are that if you are hosting an endpoint and stop the container, the endpoint remains but leads to nothing. If you restart the container, there is no guarantee that it will have the same IP address, so there is a chance of undeleted endpoints leading to the wrong app.






