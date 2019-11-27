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


and if for whatever reason it doesnt, we simply need to replace the lines in the config file that are directing our uri variable to localhost(third and fourth lines), with the following:

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

If not, double check database uri.

At this point, we should have an app running in a user-defined network.







