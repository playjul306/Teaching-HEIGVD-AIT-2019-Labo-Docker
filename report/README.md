title: Lab 04 - Docker
---

## Lab 04 - Docker


#### Pedagogical objectives

* Build your own Docker images

* Become familiar with lightweight process supervision for Docker

* Understand core concepts for dynamic scaling of an application in production

* Put into practice decentralized management of web server instances

#### Instructions for the lab report

This lab builds on a previous lab on load balancing.

In this lab you will perform a number of tasks and document your progress in a
lab report. Each task specifies one or more deliverables to be produced.
Collect all the deliverables in your lab report. Give the lab report a structure
that mimics the structure of this document.

We expect you to have in your repository (you will get the instructions later
for that) a folder called `report` and a folder called `logs`. Ideally, your
report should be in Markdown format directly in the repository.

The lab consists of 6 tasks and one initial task (the initial task
should be quick if you already completed the lab on load balancing):

0. [Identify issues and install the tools](#task-0)
1. [Add a process supervisor to run several processes](#task-1)
2. [Add a tool to manage membership in the web server cluster](#task-2)
3. [React to membership changes](#task-3)
4. [Use a template engine to easily generate configuration files](#task-4)
5. [Generate a new load balancer configuration when membership changes](#task-5)
6. [Make the load balancer automatically reload the new configuration](#task-6)

**Remarks**:

- In your report reference the task numbers and question numbers of
  this document.

- The version of HAProxy used in this lab is `1.5`. When reading the
  documentation, make sure you are looking at this version. Here is
  the link:
  <http://cbonte.github.io/haproxy-dconv/configuration-1.5.html>

  **Note**: There is an exception for that later in the lab. The
  documentation for a part of HAProxy (the command line help) is not
  available in 1.5. Therefore, we use the doc for 1.6 which is not
  different for our usage.

- In the report give the URL of the repository that you forked off this lab.

- The images and the web application are a bit different from the lab on load
  balancing. The web app no longer requires a tag. An environment variable
  is defined in the Docker files to specify a role for each image. We will see
  later how to use that.

- We expect, at least, to see in your report:

  - An introduction describing briefly the lab

  - Seven chapters, one for each task (0 to 6)

  - A table of content

  - A chapter named "Difficulties" where you describe the problems you have encountered and
    the solutions you found

  - A conclusion

**DISCLAIMER**: In this lab, we will go through one possible approach
to manage a scalable infrastructure where we can add and remove nodes
without having to rebuild the HAProxy image. This is not the only way
to achieve this goal. If you do some research you will find a lot of
tools and services to achieve the same kind of behavior.


### <a name="task-0"></a>Task 0: Identify issues and install the tools

#### Identify issues

In the previous lab, we built a simple distributed system with a load
balancer and two web applications. The architecture of our distributed
web application is shown in the following diagram:

![Architecture](assets/img/Lab4_schema.png)

The two web app containers stand for two web servers. They run a
NodeJS sample application that implements a simple REST API. Each
container exposes TCP port 3000 to receive HTTP requests.

The HAProxy load balancer is listening on TCP port 80 to receive HTTP
requests from users. These requests will be forwarded to and
load-balanced between the web app containers. Additionally it exposes
TCP ports 1936 and 9999 for the stats page and the command-line
interface.

For more details about the web application, take a look to the
[previous lab](https://github.com/SoftEng-HEIGVD/Teaching-HEIGVD-AIT-2015-Labo-02).

Now suppose you are working for a big e-tailer like Galaxus or
Zalando. Starting with Black Friday and throughout the holiday season
you see traffic to your web servers increase several times as
customers are looking for and buying presents. In January the traffic
drops back again to normal. You want to be able to add new servers as
the traffic from customers increases and you want to be able to remove
servers as the traffic goes back to normal.

Suppose further that there is an obscure bug in the web application
that the developers haven't been able to understand yet. It makes the
web servers crash unpredictably several times per week. When you
detect that a web server has crashed you kill its container and you
launch a new container.

Suppose further currently your web servers and your load balancer are
deployed like in the previous lab. What are the issues with this
architecture? Answer the following questions. The questions are
numbered from `M1` to `M6` to refer to them later in the lab. Please
give in your report the reference of the question you are answering.

1. <a name="M1"></a>**[M1]** **Do you think we can use the current**
   **solution for a production environment? What are the main problems**
   **when deploying it in a production environment?**

   **Réponse:** Non, car dans cette situation, à chaque fois qu'un serveur est déployé, il faut aller changer des fichiers de configuration dans l'infrastructure en production pour ajouter la prise en charge du nouveau noeud. Dans le cadre d'un environnement de grande envergure, ce genre de manipulation récurrente prendrait un temps conséquent à chaque ajout de noeud et donc, il faudrait automatiser cela.
   
2. <a name="M2"></a>**[M2]** **Describe what you need to do to add new**
   **`webapp` container to the infrastructure. Give the exact steps of**
   **what you have to do without modifiying the way the things are**
   **done. Hint: You probably have to modify some configuration and**
   **script files in a Docker image.**

   **Réponse:** Ajout des codes suivants dans les fichiers correspondants

   1. `docker-compose.yml`: 
   
      ```dockerfile
      services :
        webapp3:
             container_name: ${WEBAPP_3_NAME}
             build:
               context: ./webapp
               dockerfile: Dockerfile
             networks:
               heig:
                 ipv4_address: ${WEBAPP_3_IP}
             ports:
               - "4002:3000"
             environment:
                  - TAG=${WEBAPP_3_NAME}
                  - SERVER_IP=${WEBAPP_3_IP}
      
      haproxy:
             environment:
                  - WEBAPP_3_IP=${WEBAPP_3_IP}
      ```
   
   2. `\ha\scripts\run.sh`
   
      ```dockerfile
      sed -i 's/<s3>/$S3_PORT_3000_TCP_ADDR/g' /usr/local/etc/haproxy/haproxy.cfg
      ```
   
   3. `\ha\config\haproxy.cfg` 
   
      ```dockerfile
      backend nodes
      	    server s3 ${WEBAPP_3_IP}:3000 check
      ```
   
3. <a name="M3"></a>**[M3]** **Based on your previous answers, you have**
   **detected some issues in the current solution. Now propose a better**
   **approach at a high level.**

   **Réponse: **la solution serait d'avoir un agent qui tourne en background sur chaque hôte (conteneur) et qui annonçerait sa présence au load balancer en lui envoyant par exemple son adresse IP. Le load balancer pourra donc éditer son fichier de configuration et effectuer les manipulations ci-dessus pour l'ajout de l'hôte à l'environnement. 
   
4. <a name="M4"></a>**[M4]** **You probably noticed that the list of web**
    **application nodes is hardcoded in the load balancer**
    **configuration. How can we manage the web app nodes in a more dynamic**
    **fashion?**

    **Réponse: **comme discuté avec l'enseignant, la réponse à ce point est la même que le point précédent. 

5. <a name="M5"></a>**[M5]** **In the physical or virtual machines of a**
   **typical infrastructure we tend to have not only one main process**
   **(like the web server or the load balancer) running, but a few**
   **additional processes on the side to perform management tasks.**

   **For example to monitor the distributed system as a whole it is**
   **common to collect in one centralized place all the logs produced by**
   **the different machines. Therefore we need a process running on each**
   **machine that will forward the logs to the central place. (We could**
   **also imagine a central tool that reaches out to each machine to**
   **gather the logs. That's a push vs. pull problem.) It is quite**
   **common to see a push mechanism used for this kind of task.**

   **Do you think our current solution is able to run additional**
   **management processes beside the main web server / load balancer**
   **process in a container? If no, what is missing / required to reach**
   **the goal? If yes, how to proceed to run for example a log**
   **forwarding process?**

6. <a name="M6"></a>**[M6]** **In our current solution, although the**
   **load balancer configuration is changing dynamically, it doesn't**
   **follow dynamically the configuration of our distributed system when**
   **web servers are added or removed. If we take a closer look at the**
   **`run.sh` script, we see two calls to `sed` which will replace two**
   **lines in the `haproxy.cfg` configuration file just before we start**
   **`haproxy`. You clearly see that the configuration file has two**
   **lines and the script will replace these two lines.**

   **What happens if we add more web server nodes? Do you think it is**
   **really dynamic? It's far away from being a dynamic**
   **configuration. Can you propose a solution to solve this?**
   
   

**Deliverables**:

1. **Take a screenshot of the stats page of HAProxy at**
   **<http://192.168.42.42:1936>. You should see your backend nodes.**

   <img src="images/task0.png" style=" zoom:50%" />
   
2. **Give the URL of your repository URL in the lab report.**

   voici l’url de notre repo : https://github.com/playjul306/Teaching-HEIGVD-AIT-2019-Labo-Docker


### <a name="task-1"></a>Task 1: Add a process supervisor to run several processes

**Deliverables**:

1. **Take a screenshot of the stats page of HAProxy at**
   **<http://192.168.42.42:1936>. You should see your backend nodes. It**
   **should be really similar to the screenshot of the previous task.**

   <img src="images/task1.png" style="zoom:50%" />
   
2. **Describe your difficulties for this task and your understanding of**
   **what is happening during this task. Explain in your own words why**
   **are we installing a process supervisor. Do not hesitate to do more**
   **research and to find more articles on that topic to illustrate the**
   **problem.**

   Nous n’avons eu aucune difficulté lors de la réalisation de cette tâche.

   Le but de cette tâches est de permettre aux conteneurs docker d’executer plusieurs processus grâce au superviseur de processus (`s6-overlay`) ajouté aux images docker. De plus cela nous permet d’éviter que le conteneur se tue une fois un processus terminé.


### <a name="task-2"></a>Task 2: Add a tool to manage membership in the web server cluster

**Deliverables**:

1. **Provide the docker log output for each of the containers: `ha`,**
   **`s1` and `s2`. You need to create a folder `logs` in your**
   **repository to store the files separately from the lab**
   **report. For each lab task create a folder and name it using the**
   **task number. No need to create a folder when there are no logs.**

   Les fichiers de log de chaque serveur pour chaque tâche se trouvent dans le répertoire `logs/` du dossier du projet comme nous pouvons le voir sur l’image ci-dessous :

   <img src="images/task2.png" style=" zoom:75%" />
   
2. **Give the answer to the question about the existing problem with the**
   **current solution.**

   Le problème de cette solution est qu’on lie le HAProxy avec les webapp avec le flag `—link`, ce qui nous oblige à démarrer les serveurs avant le HAProxy et de devoir redémarrer le HAProxy, ainsi que tous ses noeuds pour chaque ajout de serveur.
   
3. **Give an explanation on how `Serf` is working. Read the official**
   **website to get more details about the `GOSSIP` protocol used in**
   **`Serf`. Try to find other solutions that can be used to solve**
   **similar situations where we need some auto-discovery mechanism.**

   **`Serf`** est un outil de gestion de cluster et de détection de panne. Il permet de créer un cluster avec tous ses noeuds afin qu’il puisse communiquer entre eux. Tout cela se fait grâce à l’agent serf installé sur chacun des noeuds, ce qui permet à ces derniers d’être notifié des arrivées et départs de noeuds du cluster.

   **`Serf`** permet également de détecter les noeuds défaillants en quelques secondes, d’avertir le reste du cluster et d’executer des scripts de gestion personnalisé permettant de gérer ces événements.

   **`Serf`** peut aussi propagé des événements et des requêtes personnalisés vers le cluster. Ces dernières peuvent être utilisées pour déclenché des déploiements ou encore propager des configurations.

   **`GOSSIP`** est un protocole de communication basé sur le protocole SWIM, il utilise UDP pour envoyer des messages en broadcast au cluster. C’est grâce à ce protocol Gossip que les prôblème majeur de serf (l'appartenance, la détection et la récupération des pannes et la propagation d'événements personnalisés) sont résolus.

   Autres solutions :

   - Consul
   - Etcd
   - Apache ZooKeeper

   Sources : 

   - https://www.serf.io/intro/index.html
   - https://www.serf.io/docs/internals/gossip.html
   - https://devopscube.com/open-source-service-discovery/

   


### <a name="task-3"></a>Task 3: React to membership changes

**Deliverables**:

1. **Provide the docker log output for each of the containers:  `ha`, `s1` and `s2`.**
   **Put your logs in the `logs` directory you created in the previous task.**

   Tous les logs de chaque conteneur sont dans leur répertoire réspectif (`logs/task3/<container>`).
   
2. **Provide the logs from the `ha` container gathered directly from the `/var/log/serf.log`**
   **file present in the container. Put the logs in the `logs` directory in your repo.**

   Le fichier contenant les logs de cette étape se trouve dans (`logs/task3/ha/log-ha-serf`)


### <a name="task-4"></a>Task 4: Use a template engine to easily generate configuration files

> We have to generate a new configuration file for the load balancer each time 
  a web server is added or removed. There are several ways to do this. Here we 
  choose to go the way of templates. In this task we will put in place a
  template engine and use it with a basic example. You will not become an expert
  in template engines but it will give you a taste of how to apply this technique
  which is often used in other contexts (like web templates, mail templates, ...).
  We will be able to solve the issue raised in [M6](#M6).

There are several ways to generate a configuration file from variables
in a dynamic fashion. In this lab we decided to use `NodeJS` and
`Handlebars` for the template engine.

According to Wikipedia:

  > A template engine is a software designed to combine one or more templates
    with a data model to produce one or more result documents

In our case our template is the `HAProxy` configuration file in which
we put placeholders written in the template language. Our data model
is the data provided by the handler scripts of `Serf`. And the
resulting document coming out of the template engine is a
configuration file that HA proxy can understand where the placeholders
have been replaced with the data.

**References**:

  - [NodeJS](https://nodejs.org/en/)
  - [Handlebars](http://handlebarsjs.com/)
  - [Template Engine definition](https://en.wikipedia.org/wiki/Template_processor)

To be able to use `Handlebars` as a template engine in our `ha`
container, we need to install `NodeJS` and `Handlebars`.

To install `NodeJS`, just replace `TODO: [HB] Install NodeJS` by the
following content:

```
# Install NodeJS
RUN curl -sSLo /tmp/node.tar.xz https://nodejs.org/dist/v4.4.4/node-v4.4.4-linux-x64.tar.xz \
  && tar -C /usr/local --strip-components 1 -xf /tmp/node.tar.xz \
  && rm -f /tmp/node.tar.xz
```

We also need to update the base tools installed in the image to be
able to extract the `NodeJS` archive. So we need to add `xz-utils` to
the `apt-get install` present above the line `TODO: [HB] Update to
install required tool to install NodeJS`.

**Remarks**:

  - You probably noticed that we have the webapp image with a `NodeJS`
    application.  So the image already contains `NodeJS`. We have
    based our backend image on an existing image that provides an
    installation of `NodeJS`. In our `ha` image, we take a shortcut
    and do a manual installation of `NodeJS`.

    This manual install has at least one bad practice: In the original
    image of `NodeJS` they download of the required files and then
    check the downloads against a `GPG` signatures. We have skipped
    this part in our `ha`image, but in practice you should check every
    download to avoid issues like the `man in the middle` attack.

    You can take a look at the following links if you are interested
    in this topic:

      - [NodeJS official Dockerfile](https://github.com/nodejs/docker-node/blob/ae9e2d4f04a0fa82261df86fd9556a76cefc020d/6.3/wheezy/Dockerfile#L4-L26)
      - [GPG](https://en.wikipedia.org/wiki/GNU_Privacy_Guard)
      - [Man in the middle attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)

    The other reason why we have to manually install `NodeJS` is that
    we cannot inherit from two images at the same time. As in our `ha`
    image we already inherit `FROM` the `haproxy` official image we
    cannot use the `NodeJS` image at the same time.

    In fact, the `FROM` instruction from Docker works like the Java
    inheritance model. You can inherit only from one super class at a
    time. For example, we have the following hierarchy for our HAProxy
    image.

    <a href="https://github.com/SoftEng-HEIGVD/Teaching-HEIGVD-AIT-2016-Labo-Docker/blob/master/assets/img/image-hierarchy.png">
      <img src="https://github.com/SoftEng-HEIGVD/Teaching-HEIGVD-AIT-2016-Labo-Docker/raw/master/assets/img/image-hierarchy.png" alt="HAProxy Image Hierarchy" width="600">
    </a>

    Here is the reference to the Docker documentation of the `FROM` command:

      - [FROM](https://docs.docker.com/engine/reference/builder/#/from)

It's time to install `Handlebars` and a small command line tool
`handlebars-cmd` to make it work properly. For that replace the `TODO:
[HB] Install Handlebars and cli` by this Docker instruction:

```
# Install the handlebars-cmd node module and its dependencies
RUN npm install -g handlebars-cmd
```

**Remarks**:

  - [NPM](http://npmjs.org/) is a package manager for `NodeJS`. Like
    other package managers, one of its tasks is to manage the
    dependencies of a package. That's the reason why we have to
    install only `handlebars-cmd`. This package has the `handlebars`
    package as one of its dependencies.

Now we will update the handler scripts to use `Handlebars`. For the moment, we
will just play with a simple template. So, first create a file in `ha/config` called
`haproxy.cfg.hb` with a simple template content. Use the following command for that:

```bash
echo "Container {{ name }} has joined the Serf cluster with the following IP address: {{ ip }}" >> /ha/config/haproxy.cfg.hb
```

We need our template present in our `ha` image. We have to add the following
Docker instructions for that. Let's replace `TODO: [HB] Copy the haproxy configuration template`
in [ha/Dockerfile](ha/Dockerfile#L32) with the required stuff to:

  1. Have a directory `/config`
  2. Have the `haproxy.cfg.hb` in it

Then, update the `member-join.sh` script in [ha/scripts](ha/scripts) with the following content:

```bash
#!/usr/bin/env bash

echo "Member join script triggered" >> /var/log/serf.log

# We iterate over stdin
while read -a values; do
  # We extract the hostname, the ip, the role of each line and the tags
  HOSTNAME=${values[0]}
  HOSTIP=${values[1]}
  HOSTROLE=${values[2]}
  HOSTTAGS=${values[3]}

  echo "Member join event received from: $HOSTNAME with role $HOSTROLE" >> /var/log/serf.log

  # Generate the output file based on the template with the parameters as input for placeholders
  handlebars --name $HOSTNAME --ip $HOSTIP < /config/haproxy.cfg.hb > /tmp/haproxy.cfg
done
```

<a name="ttb"></a>
Time to build our `ha` image and run it. We will also run `s1` and `s2`. As usual, here
are the commands to build and run our image and containers:

```bash
# Remove running containers
docker rm -f ha s1 s2

# Build the haproxy image
cd ha
docker build -t <imageName> .

# Run the HAProxy container
docker run -d -p 80:80 -p 1936:1936 -p 9999:9999 --network heig --name ha <imageName>

# OR
docker-compose up --build
```

**Remarks**:

  - Installing a new util with `apt-get` means building the whole image again as
    it is in our Docker file. This will take few minutes.

Take the time to retrieve the output file in the `ha` container. Connect to the container:

```bash
docker exec -ti ha /bin/bash
```

and get the content from the file (**keep it for deliverables, handle it as you do for the logs**)

```bash
cat /tmp/haproxy.cfg
```

After you have inspected the generated file quit the container with `exit`.

Now that we invoke the template engine from the handler script it is
time to do an end-to-end test. Start the `s1` container, wait a bit,
then retrieve the `haproxy.cfg` file from the `ha` container to see
whether it saw `s1` coming up. Then do the same for `s2`:

```bash
# 1) Run the S1 container
docker run -d --network heig --name s1 <imageName>

# 2) Connect to the ha container (optional if you have another ssh session)
docker exec -ti ha /bin/bash

# 3) From the container, extract the content (keep it for deliverables)
cat /tmp/haproxy.cfg

# 4) Quit the ha container (optional if you have another ssh session)
exit

# 5) Run the S2 container
docker run -d --network heig --name s2 <imageName>

# 6) Connect to the ha container (optional if you have another ssh session)
docker exec -ti ha /bin/bash

# 7) From the container, extract the content (keep it for deliverables)
cat /tmp/haproxy.cfg

# 8) Quit the ha container
exit
```

**Deliverables**:

1. You probably noticed when we added `xz-utils`, we have to rebuild
   the whole image which took some time. What can we do to mitigate
   that? Take a look at the Docker documentation on
   [image layers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/#images-and-layers).
   Tell us about the pros and cons to merge as much as possible of the
   command. In other words, compare:

  ```
  RUN command 1
  RUN command 2
  RUN command 3
  ```

  vs.

  ```
  RUN command 1 && command 2 && command 3
  ```

  There are also some articles about techniques to reduce the image
  size. Try to find them. They are talking about `squashing` or
  `flattening` images.

2. Propose a different approach to architecture our images to be able
   to reuse as much as possible what we have done. Your proposition
   should also try to avoid as much as possible repetitions between
   your images.

3. Provide the `/tmp/haproxy.cfg` file generated in the `ha` container
   after each step.  Place the output into the `logs` folder like you
   already did for the Docker logs in the previous tasks. Three files
   are expected.
   
   In addition, provide a log file containing the output of the 
   `docker ps` console and another file (per container) with
   `docker inspect <container>`. Four files are expected.
   
4. Based on the three output files you have collected, what can you
   say about the way we generate it? What is the problem if any?


### <a name="task-5"></a>Task 5: Generate a new load balancer configuration when membership changes

> We now have S6 and Serf ready in our HAProxy image. We have member
  join/leave handler scripts and we have the handlebars template
  engine. So we have all the pieces ready to generate the HAProxy
  configuration dynamically. We will update our handler scripts to
  manage the list of nodes and to generate the HAProxy configuration
  each time the cluster has a member leave/join event.  The work in
  this task will let us solve the problem mentioned in [M4](#M4).

At this stage, we have:

  - Two images with `S6` process supervisor that starts a Serf agent
    and an "application" (HAProxy or Node web app).

  - The `ha` image contains the required stuff to react to `Serf`
    events when a container joins or leaves the `Serf` cluster.

  - A template engine in the `ha` image is ready to be used to
    generate the HAProxy configuration file.

Now, we need to refine our `join` and `leave` scripts to generate a
proper HAProxy configuration file.

First, we will copy/paste the content of the
[ha/config/haproxy.cfg](ha/config/haproxy.cfg) file into the template
[ha/config/haproxy.cfg.hb](ha/config/haproxy.cfg.hb). You can simply
run the following command:

```bash
cp ha/config/haproxy.cfg ha/config/haproxy.cfg.hb
```

Then we will replace the content between `# HANDLEBARS START` and
`# HANDLEBARS STOP` by the following content:

```
{{#each addresses}}
server {{ host }} {{ ip }}:3000 check
{{/each}}
```

**Remarks**:

  - `each` iterates over a collection of data

  - `{{` and `}}` are the bars that will be interpreted by `handlebars`

  - `host` and `ip` are the data contained in the JSON format of the collection
    that handlebars will receive. We will see that right after in the `member-join.sh`
    script. The JSON format will be: `{ "host": "<hostname>", "ip": "<ip address>" }`.

Our configuration template is ready. Let's update the `member-join.sh` script to
generate the correct configuration.

The mechanism to manage the `join` and `leave` events is the following:

  1. We check if the event comes from a backend node (the role is used).

  2. We create a file with the hostname and IP address of each backend
     node that joins the cluster.

  3. We build the `handlebars` command to generate the new configuration from the list
     of files that represent our backend nodes

The same logic also applies when a node leaves the cluster. In this
case, the second step will remove the file with the node data.

In the file [ha/scripts/member-join.sh](ha/scripts/member-join.sh)
replace the whole content by the following one. Take the time to read the comments.

```bash
#!/usr/bin/env bash

echo "Member join script triggered" >> /var/log/serf.log

BACKEND_REGISTERED=false

# We iterate over stdin
while read -a values; do
  # We extract the hostname, the ip, the role of each line and the tags
  HOSTNAME=${values[0]}
  HOSTIP=${values[1]}
  HOSTROLE=${values[2]}
  HOSTTAGS=${values[3]}

  # We only register the backend nodes
  if [[ "$HOSTROLE" == "backend" ]]; then
    echo "Member join event received from: $HOSTNAME with role $HOSTROLE" >> /var/log/serf.log

    # We simply register the backend IP and hostname in a file in /nodes
    # with the hostname for the file name
    echo "$HOSTNAME $HOSTIP" > /nodes/$HOSTNAME

    # We have at least one new node registered
    BACKEND_REGISTERED=true
  fi
done

# We only update the HAProxy configuration if we have at least one new  backend node
if [[ "$BACKEND_REGISTERED" = true ]]; then
  # To build the collection of nodes
  HOSTS=""

  # We iterate over each backend node registered
  for hostfile in $(ls /nodes); do
    # We convert the content of the backend node file to a JSON format: { "host": "<hostname>", "ip": "<ip address>" }
    CURRENT_HOST=`cat /nodes/$hostfile | awk '{ print "{\"host\":\"" $1 "\",\"ip\":\"" $2 "\"}" }'`

    # We concatenate each host
    HOSTS="$HOSTS$CURRENT_HOST,"
  done

  # We process the template with handlebars. The sed command will simply remove the
  # trailing comma from the hosts list.
  handlebars --addresses "[$(echo $HOSTS | sed s/,$//)]" < /config/haproxy.cfg.hb > /usr/local/etc/haproxy/haproxy.cfg

  # TODO: [CFG] Add the command to restart HAProxy
fi
```

And here we go for the `member-leave.sh` script. The script differs only for the part where
we remove the backend nodes registered via the `member-join.sh`.

```bash
#!/usr/bin/env bash

echo "Member leave/join script triggered" >> /var/log/serf.log

BACKEND_UNREGISTERED=false

# We iterate over stdin
while read -a values; do
  # We extract the hostname, the ip, the role of each line and the tags
  HOSTNAME=${values[0]}
  HOSTIP=${values[1]}
  HOSTROLE=${values[2]}
  HOSTTAGS=${values[3]}

  # We only remove the backend nodes
  if [[ "$HOSTROLE" == "backend" ]]; then
    echo "Member $SERF_EVENT event received from: $HOSTNAME with role $HOSTROLE" >> /var/log/serf.log

    # We simply remove the file that was used to track the registered node
    rm /nodes/$HOSTNAME

    # We have at least one new node that leave the cluster
    BACKEND_UNREGISTERED=true
  fi
done

# We only update the HAProxy configuration if we have at least a backend that
# left the cluster. The process to generate the HAProxy configuration is the
# same than for the member-join script.
if [[ "$BACKEND_UNREGISTERED" = true ]]; then
  # To build the collection of nodes
  HOSTS=""

  # We iterate over each backend node registered
  for hostfile in $(ls /nodes); do
    # We convert the content of the backend node file to a JSON format: { "host": "<hostname>", "ip": "<ip address>" }
    CURRENT_HOST=`cat /nodes/$hostfile | awk '{ print "{\"host\":\"" $1 "\",\"ip\":\"" $2 "\"}" }'`

    # We concatenate each host
    HOSTS="$HOSTS$CURRENT_HOST,"
  done

  # We process the template with handlebars. The sed command will simply remove the
  # trailing comma from the hosts list.
  handlebars --addresses "[$(echo $HOSTS | sed s/,$//)]" < /config/haproxy.cfg.hb > /usr/local/etc/haproxy/haproxy.cfg

  # TODO: [CFG] Add the command to restart HAProxy
fi
```

**Remarks**:

  - The way we keep track the backend nodes is pretty simple and makes
    the assumption there is no concurrency issue with `Serf`. That's
    reasonable enough to get a quite simple solution.

**Cleanup**:

  - In the main configuration file that is used for bootstrapping
    HAProxy the first time when there are no backend nodes, we have
    the list of servers that we used in the first task and the
    previous lab. We can remove the list. So find `TODO: [CFG] Remove
    all the servers` and remove the list of nodes.

  - In [ha/services/ha/run](ha/services/ha/run), we can remove the two lines
    above `TODO: [CFG] Remove the following two lines`.

We need to make sure the image has the folder `/nodes` created. In the
Docker file, replace the `TODO: [CFG] Create the nodes folder` by the
correct instruction to create the `/nodes` folder.

We are ready to build and test our `ha` image. Let's proceed like in
the [previous task](#ttb).  You should provide the same outputs for
the deliverables. Remember that we have moved the file
`/tmp/haproxy.cfg` to `/usr/local/etc/haproxy/haproxy.cfg` (**keep
track of the config file like in previous step and also the output of
`docker ps` and `docker inspect <container>`**).

You can also get the list of registered nodes from inside the `ha`
container. Simply list the files from the directory `/nodes`.  (**keep
track of the output of the command like the logs in previous tasks**)

Now, use the Docker commands to stop `s1`.

You can connect again to the `ha` container and get the haproxy
configuration file and also the list of backend nodes. Use the
previous command to reach this goal.  (**keep track of the output of
the ls command and the configuration file like the logs in previous
tasks**)

**Deliverables**:

1. Provide the file `/usr/local/etc/haproxy/haproxy.cfg` generated in
   the `ha` container after each step. Three files are expected.
   
   In addition, provide a log file containing the output of the 
   `docker ps` console and another file (per container) with
   `docker inspect <container>`. Four files are expected.

2. Provide the list of files from the `/nodes` folder inside the `ha` container.
   One file expected with the command output.

3. Provide the configuration file after you stopped one container and
   the list of nodes present in the `/nodes` folder. One file expected
   with the command output. Two files are expected.
   
    In addition, provide a log file containing the output of the 
   `docker ps` console. One file expected.

4. (Optional:) Propose a different approach to manage the list of backend
   nodes. You do not need to implement it. You can also propose your
   own tools or the ones you discovered online. In that case, do not
   forget to cite your references.

### <a name="task-6"></a>Task 6: Make the load balancer automatically reload the new configuration

> Finally, we have all the pieces in place to finish our
  solution. HAProxy will be reconfigured automatically when web app
  nodes are leaving/joining the cluster. We will solve the problems
  you have discussed in [M1 - 3](#M1).  Again, the solution built
  in this lab is only one example of tools and techniques we can use to
  solve this kind of situation. There are several other ways.

The only thing missing now is to make sure the configuration of
HAProxy is up-to-date and taken into account by HAProxy.

We will try to make HAProxy reload his config with minimal
downtime. At the moment, we will replace the line `TODO: [CFG] Replace
this command` in [ha/services/ha/run](ha/services/ha/run) by the
following script part. As usual, take the time to read the comments.

```bash
#!/usr/bin/with-contenv bash

# ##############################################################################
# WARNING
# ##############################################################################
# S6 expects the processes it manages to stop when it sends them a SIGTERM signal.
# The Serf agent does not stop properly when receiving a SIGTERM signal.
#
# Therefore, we need to do some tricks to remedy the situation. We need to
# "simulate" the handling of SIGTERM in the script and send to Serf the signal
# that makes it quit (SIGINT).
#
# Basically we need to do the following:
# 1. Keep track of the process id (PID) of Serf Agent
# 2. Catch the SIGTERM from S6 and send a SIGINT to Serf
# 3. Make sure this shell script will not stop before S6 stops it, but when
#    SIGTERM is sent, we need to stop everything.

# Get the current process ID to avoid killing an unwanted process
pid=$$

# Define a function to kill the Serf process as Serf does not accept SIGTERM. In
# place, we will send a SIGINT signal to the process to stop it correctly.
sigterm() {
  kill -USR1 $pid
}

# Trap the SIGTERM and in place run the function that will kill the process
trap sigterm SIGTERM

# We need to keep track of the PID of HAProxy in a file for the restart process.
# We are forced to do that because the blocking process for S6 is this shell
# script. When we send to S6 a command to restart our process, we will lose
# the value of the variable pid. The pid variable will stay alive until any
# restart or stop from S6.
#
# In the case of a restart we need to keep the HAProxy PID to give it back to
# HAProxy. The comments on the HAProxy command will complete this exaplanation.
if [ -f /var/run/haproxy.pid ]; then
    HANDOFFPID=`cat /var/run/haproxy.pid`
fi

# To kill an old HAProxy and start a new one with minimal outage 
# HAProxy provides the -sf/-st command-line options. With these options 
# one can give the PIDs of currently running HAProxy processes at startup.
# This will start new HAProxy processes and when startup is complete
# it send FINISH or TERMINATE signals to the ones given in the argument. 
#
# The HANDOFFPID keeps track of the PID of HAProxy. We retrieve it from the
# the file we written the last time we (re)started HAProxy.
exec haproxy -f /usr/local/etc/haproxy/haproxy.cfg -sf $HANDOFFPID &

# Retrieve the process ID of the command run in background. Doing that, we will
# be able to send the SIGINT signal through the sigterm function we defined
# to replace the SIGTERM.
pid=$!

# And write it to a file to get it on next restart
echo $pid > /var/run/haproxy.pid

# Finally, we wait as S6 launches this shell script. This will simulate
# a foreground process for S6. All that tricky stuff is required because
# we use a process supervisor in a Docker environment. The applications need
# to be adapted for such environments.
wait
```

**Remarks**:

  - In this lab, we do not achieve an HAProxy restart with _zero_
    downtime. You will find an article about that in the references.

**References**:

  - [Stopping HAProxy](http://cbonte.github.io/haproxy-dconv/1.6/management.html#4)
  - [Sending signal to Processes](https://bash.cyberciti.biz/guide/Sending_signal_to_Processes)
  - [Zero downtime with HAProxy article](http://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html)

We need to update our `member-join` and `member-leave` scripts to make sure HAProxy
will be restarted when its configuration is modified. For that, in both files, replace
`TODO: [CFG] Add the command to restart HAProxy` by the following command.

```bash
# Send a SIGHUP to the process. It will restart HAProxy
s6-svc -h /var/run/s6/services/ha
```

**References**:

  - [S6 svc doc](http://skarnet.org/software/s6/s6-svc.html)

It's time to build and run our images. At this stage, if you try to reach
`http://192.168.42.42`, it will not work. No surprise as we do not start any
backend node. Let's start one container and try to reach the same URL.

You can start the web application nodes. If everything works well, you could
reach your backend application through the load balancer.

And now you can start and stop any number of nodes you want! You will
see the dynamic reconfiguration occurring. Keep in mind that HAProxy
will take few seconds before nodes will be available. The reason is
that HAProxy is not so quick to restart inside the container and your
web application is also taking time to bootstrap. And finally,
depending of the health checks of HAProxy, your web app will not be
available instantly.

Finally, we achieved our goal to build an architecture that is dynamic
and reacts to nodes coming and going!

![Final architecture](assets/img/Lab4_schemaSerf.png)

**Deliverables**:

1. Take a screenshots of the HAProxy stat page showing more than 2 web
   applications running. Additional screenshots are welcome to see a
   sequence of experimentations like shutting down a node and starting
   more nodes.
   
   Also provide the output of `docker ps` in a log file. At least 
   one file is expected. You can provide one output per step of your
   experimentation according to your screenshots.
   
2. Give your own feelings about the final solution. Propose
   improvements or ways to do the things differently. If any, provide
   references to your readings for the improvements.

3. (Optional:) Present a live demo where you add and remove a backend container.


## Windows troubleshooting

It appears that Windows users can encounter a `CRLF` vs. `LF` problem when the repos is cloned without taking care of the ending lines. Therefore, if the ending lines are `CRFL`, it will produce an error message with Docker: 

```bash
... no such file or directory
```

(Take a look to this Docker issue: https://github.com/docker/docker/issues/9066, the last post show the error message).

The error message is not really relevant and difficult to troubleshoot. It seems the problem is caused by the line endings not correctly interpreted by Linux when they are `CRLF` in place of `LF`. The problem is caused by cloning the repos on Windows with a system that will not keep the `LF` in the files.

Fortunatelly, there is a procedure to fix the `CRLF` to `LF` and then be sure Docker will recognize the `*.sh` files.

First, you need to add the file `.gitattributes` file with the following content:

```bash
* text eol=lf
```

This will ask the repos to force the ending lines to `LF` for every text files.

Then, you need to reset your repository. Be sure you do not have **modified** files.

```bash
# Erease all the files in your local repository
git rm --cached -r .

# Restore the files from your local repository and apply the correct ending lines (LF)
git reset --hard
```

Then, you are ready to go.

There is a link to deeper explanation and procedure about the ending lines written by GitHub: https://help.github.com/articles/dealing-with-line-endings/