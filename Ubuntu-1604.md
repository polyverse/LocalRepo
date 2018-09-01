
# Local Polyverse Repository (subset)
This guide walks through each step of creating a subset of the [Polymorphic Linux](https://polyverse.io) scrambled package repository. Since the targeted client environment only has a small set of packages that it will actually be using, the local repository only needs to perform a daily retrieval of the files in that list from the Polyverse scrambled binary repository.

## Containers

### 1. Setup Requirements
A minimum of two containers is required for this example to run properly. In this example, we will use three containers, running Ubuntu 16.04 (Xenial), for a very clear example.
* One container (**CustomRepository**) will be used as the local package repository
* One container (**Client1**) will be used as a standard client retrieving packages from the aforementioned package repository
* One container (**Client2**) will be used as a standard client retrieving packages from the aforementioned package repository
* Any additional number of clients can be configured to consume from the repository, but they must be modeled after the final configuration of **Client2**.

### 2. Retrieve the required list of packages that the mirror must continually rotate

* Run the **Client1** container in the background
    ```sh
    $ docker run -dt --rm --name=Client1 polyverse/example-isolated-client-ubuntu sh
    ```

* Open an interactive terminal session for you to interact with the **Client1** container
    ```sh
    $ docker exec -it Client1 bash
    ```

* In the **Client1** container:

    * Output the list of installed packages as a consumable string by using the following command:
        ```sh
        $ apt list --installed | awk -F'/' '{print $1}' | grep -v Listing | xargs
        ```
        * Copy the output from the above command into your clipboard, and paste it into a file called **InstalledPackages.txt** for safe-keeping somewhere outside of the container.
        We will use this list of packages later in the guide as our authoritative list for the packages that will be regularly rotated in the Custom Repository container

    * Exit the container
        ```sh
        $ exit
        ```

### 3. Setup the custom repository for consumption
* Run the **CustomRepository** container in the background
    ```sh
    $ docker run -dt --rm --name=CustomRepository -p 80:80 ubuntu:16.04 sh
    ```

* Open an interactive terminal session for you to interact with the **CustomRepository** container
    ```sh
    $ docker exec -it CustomRepository bash
    ```

* In the **CustomRepository** container

    * Install and Configure [Polymorphic Linux](https://polyverse.io)
    
        * Update the container’s packages and apt’s cache
            ```sh
            $ apt-get update
            ```
            
        * Download curl so that the [Polymorphic Linux](https://polyverse.io) installation script can be used
            ```sh
            $ apt-get -y install curl
            ```
            
        * Run the [Polymorphic Linux](https://polyverse.io) installation script
            ```sh
            $ curl -s https://sh.polyverse.io | sh -s install <your_polyverse_auth_key_here>
            ```
            
        * Update APT to retrieve the new index file from [Polymorphic Linux](https://polyverse.io)
            ```sh
            $ apt-get update
            ```
    
    * Setup the environment for retrieving the specific client packages that it will serve

        * Create a folder where you will cache the scrambled version of the packages that are needed by the client devices. We will refer to that folder as /polyverse/client_packages within the context of this guide.
            ```sh
            $ mkdir -p /polyverse/client_packages/downloaded/amd64
            ```
        
        * Create a blank PackagesToDownload.txt file in /polyverse/client_packages
            ```sh
            $ cd /polyverse/client_packages
            $ touch PackagesToDownload.txt
            ```
        
        * Find your **InstalledPackages.txt file** (from earlier in the guide), and copy the contents of the file to your clipboard
        
        * Insert the package names from your clipboard into the file in the container
            ```sh
            $ echo "<paste_or_type_contents_of_InstalledPackages.txt>" > /polyverse/client_packages/PackagesToDownload.txt
            ```
    
    * Download the required packages
    *NOTE: These steps should be repeated every 24 hours.*

        * Navigate into the /polyverse/client_packages/downloaded/amd64 folder
            ```sh
            $ cd /polyverse/client_packages/downloaded/amd64
            ```
            
        * Download the packages into the current folder (should be /polyverse/client_packages/downloaded/amd64)
            ```sh
            $ apt-get download $(cat /polyverse/client_packages/PackagesToDownload.txt | xargs)
            ```
            
        * Create a catalog for APT clients to recognize and consume
            ```sh
            $ apt-get -y install dpkg-dev
            $ cd /polyverse/client_packages/downloaded/
            $ dpkg-scanpackages amd64 | gzip -9c > amd64/Packages.gz
            ```
    
    * Configure the web server
    
        * Install the web server, which will host our repository for client consumption. In this guide, we will be using Apache.
            ```sh
            $ apt-get install -y apache2
            $ mkdir -p /var/www/html/polyverse
            $ ln -s /polyverse/client_packages/downloaded /var/www/html/polyverse/scrambled-packages
            ```
        
        * Run the web server
            ```sh
            $ service apache2 start
            ```
        
    * Congratulations! The system is now setup to serve packages to clients without those clients needing to connect to any other source for publicly available packages.

## Existing and New Client Setup
For this example, since we already have a **Client1** running, we will use **Client2** for our new container.

1. Run a new Client2 container in the background
    ```sh
    $ docker run -dt --rm --name=Client2 ubuntu:16.04 sh
    ```

2. Open an interactive terminal session for you to interact with the Client2 container
    ```sh
    $ docker exec -it Client2 bash
    ```
    
3. Backup the current sources.list file
    ```sh
    $ cp /etc/apt/sources.list /etc/apt/sources.list.pvbakup
    ```

4. Set the network location of **CustomRepository**  as the only source from which **Client2** can retrieve packages
In this guide, we are using Docker containers, and our **CustomRepository** is another container. This means that we can use docker.for.mac.localhost to identify our current machine as the host.
You should use whatever the network location for the machine where your custom repository is setup.
    ```sh
    $ echo "deb http://docker.for.mac.localhost/polyverse/scrambled-packages amd64/" > /etc/apt/sources.list
    ```

5. Update APT to retrieve the new index file from the local **CustomRepository** location
    ```sh
    $ apt-get update
    ```

6. If you are on an existing client that already has all of its packages installed:
    * Reinstall all of the currently installed packages so that all installed packages will be scrambled
        ```sh
        $ apt-get -y install --reinstall $(dpkg --get-selections | awk '{print $1}')
        ```
    * If you receive an authentication error, use the `--allow-unauthenticated` flag
        ```sh
        $ apt-get -y install --reinstall $(dpkg --get-selections | awk '{print $1}') --allow-unauthenticated
        ```

7. If you are on a new, vanilla client that does not yet have the required packages installed:
    * Install any packages that are needed using standard
        ```sh
        apt-get install <package_name>
        ```
        * Note: If you receive an authentication error, use the `--allow-unauthenticated` flag
            ```sh
            $ apt-get -y install --allow-unauthenticated <package_name>
            ```
