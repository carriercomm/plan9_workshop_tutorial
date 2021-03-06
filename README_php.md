Tutorial for the DECK36 Plan9 Workshop using PHP
================================================
[1]: https://www.virtualbox.org/wiki/Downloads "Virtual Box"
[2]: http://www.vagrantup.com/downloads.html "Vagrant"
[3]: https://github.com/DECK36/plan9_workshop_tutorial/archive/master.zip "Workshop Repo"
[4]: https://github.com/DECK36/deck36-php-web-app "Plan9 Web Application"
[5]: https://github.com/DECK36/deck36-api-backend "Plan9 NodeJS API Backend"
[6]: https://github.com/DECK36/deck36-storm-backend-php "Plan9 Storm Backend with PHP"
[7]: http://www.jetbrains.com/idea/download "IntelliJ Download"
[8]: https://api.plan9-dev.deck36.de "Api dev"
[9]: https://www.plan9-dev.deck36.de "Plan9 dev"
[DEVMD]: ./DEV_php.md "Devmd"
[README]: ./README.md "README"
[logo]: ./deck36.png "Deck36 Logo"

![Deck36 Logo][logo]

## Introduction
This tutorial is dedicated to the workshops held by DECK36.

## Installation
The workshop assumes the p9fok VM to be up and running and the sources mounted in the VM.

See the [main README][README] for instructions to setup the dev environment. 

## Fetch Sources and Build Subprojects
1. Enter the plan9_workshop_tutorial-master/source/plan9 directory

    ```
    cd  plan9_workshop_tutorial-master/source/plan9
    ```
    
2. Clone the [web application][4]
    
    ```
    git clone https://github.com/DECK36/deck36-php-web-app
    ```

3. Install the php-web-app
    Follow the [README.md][4].

4. Clone the [nodejs api backend][5]
    
    ```
    git clone https://github.com/DECK36/deck36-api-backend
    ```

5. Install the nodejs api backend
    Follow the [README.md][5].

6. Clone the [storm backend][6]
    
    ```
    git clone https://github.com/DECK36/deck36-storm-backend-php
    ```

7. Install the storm backend
    Follow the [README.md][6].

## Start all components
1. Enter the virtual mashine
   
    ```
    vagrant ssh
    ```

2. Got to the plan9 source dir in the vm
    
    ```
    cd plan9
    ```

3. Start all services
    
    ```
    cd ~/plan9/deck36-api-backend;
    grunt start-plan9-dev;
    ```

## Play Plan9 From Outer Space
1. Go to the [https://api.plan9-dev.deck36.de][8] and [https://www.plan9-dev.deck36.de][9] domains for dev and accept the certificates
2. Open [https://www.plan9-dev.deck36.de][9] and register a user (or login with the one created on cli)
3. Go to the playground and play

## Add your own Badge!
[Follow the dev instructions][DEVMD]
