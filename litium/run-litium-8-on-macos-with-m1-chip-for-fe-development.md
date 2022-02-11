# Run Litium 8 on macOS with M1 chip for FE development

1. Download and install [.NET 6](https://dotnet.microsoft.com/en-us/download/dotnet/6.0)

1. Install Litium nuget package

    ```bash
    dotnet nuget add source https://nuget.litium.com/nuget/ -n Litium -u <USERNAME> -p <PASSWORD> --store-password-in-clear-text
    ```

    Litium documentation account can be [created here](https://docs.litium.com/system_pages/login_ls).

1. Install Docker

    Download it [here for M1 chip](https://docs.docker.com/desktop/mac/apple-silicon/)

    Start the Docker app

    ```bash
    open /Applications/Docker.app
    ```

    Then add the Litium registry to Docker

    ```bash
    docker docker login registry.litium.cloud -u <USERNAME>
    ```

1. Update the hosts file to make it possible for Docker to work on both host and the container

    ```bash
    sudo nano /etc/hosts
    ```

    Add or modify the following lines

    ```bash
    127.0.0.1 kubernetes.docker.internal
    <YOUR_COMPUTER_IP> host.docker.internal
    <YOUR_COMPUTER_IP> gateway.docker.internal
    ```



1. Install Rosetta 2

    To make sure that it is possible to run some older apps:

    ```bash
    softwareupdate --install-rosetta
    ```

1. Docker developer certificate

    Create and install certificate for https connections by running the following commands

    ```bash
    # 1 - Get the localhost.config file
    mkdir -p ~/.litium && curl -L https://raw.githubusercontent.com/LitiumAB/Education/main/Developer%20Education/Tasks/Developer%20certificate/Resources/localhost.config -o ~/.litium/localhost.config
    
    # 2 - Step into folder
    cd ~/.litium

    # 3 - Generate certificate file and key file
    docker run --rm -it -v $(pwd):/data:rw alpine/openssl \
        req -x509 \
        -nodes \
        -days 3650 \
        -newkey rsa:4096 \
        -keyout /data/generated-localhost.key \
        -out /data/generated-localhost.crt \
        -config /data/localhost.config \
        -subj "/CN=localhost"

    # 4 - Generate random password
    _P=$(openssl rand -hex 8)

    # 5 - Export a .pfx containing the certificate and is protected by the the password
    docker run --rm -it -v $(pwd):/data:rw alpine/openssl \
        pkcs12 -export \
        -out /data/generated-localhost.pfx \
        -inkey /data/generated-localhost.key \
        -in /data/generated-localhost.crt \
        -passout pass:$_P

    # 6 - Import the .pfx file to the user's certificates and add to list of trusted certificates
    dotnet dev-certs https --clean && \
    dotnet dev-certs https -ep $(pwd)/generated-localhost.pfx --password $_P --trust
    
    # 7 - Do some cleanup
    rm -rf generated*
    ```

1. Copy the content below and put in `<PROJECT_DIR>/.devsupport/docker-compose-mac.yml` under the root of you project. Can be included in version control.

    The synonym server, direct shipment and direct payment servers are deactivated to make it work on a Mac with M1 chip. Other versions are bumped and the sql server is replaced to use *azure-sql-edge* instead of *mssql*.

    ```bash
    # Docker compose below sets up the containers needed to run Litium locally:

    version: '3'
    services:

    dnsresolver:
        # https://github.com/cytopia/docker-bind
        image: cytopia/bind
        container_name: dnsresolver
        ports:
        - "53:53/tcp"
        - "53:53/udp"
        environment: 
        - WILDCARD_DNS=localtest.me=host.docker.internal
        - DNS_FORWARDER=192.168.65.5
        dns: 192.168.65.5
        restart: unless-stopped
    
    elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
        container_name: elasticsearch
        depends_on:
        - dnsresolver
        dns: 192.168.65.2
        restart: unless-stopped
        ports:
        - "9200:9200"
        environment:
        - discovery.type=single-node
        # Allocate 2GB RAM instead of the default 512MB
        # comment out the line below for additional memory allocation
        # - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
        volumes:
        - ./volumes/elasticsearch/data:/usr/share/elasticsearch/data
        entrypoint: 
        - /bin/sh
        - -c
        # The accelerator implementation of Elasticsearch require the analysis-dynamic-synonym.
        # The plugin refreshes the list of synonyms in Elasticsearch every minute allowing synonyms 
        # to be added/modified in Litium backoffice and updated in Elasticsearch without downtime.
        - "./bin/elasticsearch-plugin list | grep -q analysis-dynamic-synonym || ./bin/elasticsearch-plugin install -b https://github.com/Tasteful/elasticsearch-analysis-dynamic-synonym/releases/download/v7.6.2/elasticsearch-analysis-dynamic-synonym.zip; /usr/local/bin/docker-entrypoint.sh"
    
    # synonymserver:
    #   # Synonym server to provide elasticsearch with synonyms.
    #   image: registry.litium.cloud/apps/synonym-server:1.2.0
    #   platform: linux/amd64
    #   container_name: synonymserver
    #   restart: unless-stopped
    #   ports:
    #   - "9210:80"
    #   environment:
    #   - DataFolder=/app_data
    #   volumes:
    #   - ./volumes/synonymserver/data:/app_data

    kibana:
        # The Kibana image tries, by default, to connect to a host/container called elasticsearch.
        image: docker.elastic.co/kibana/kibana:7.17.0
        container_name: kibana
        depends_on:
        - elasticsearch
        restart: unless-stopped
        ports:
        - "5601:5601"

    redis:
        image: redis:5.0.5-alpine
        container_name: redis
        restart: unless-stopped
        ports:
        - "6379:6379"

    sqlserver:
        image: mcr.microsoft.com/azure-sql-edge
        container_name: sqlserver
        environment:
        - MSSQL_SA_PASSWORD=Pass@word
        - ACCEPT_EULA=Y
        restart: unless-stopped
        ports:
        # Make the SQL Container available on port 5434 to not conflict with a previously installed local SQL instance.
        # If you do not have SQL Server installed you can use 1433:1433 as mapping and skip port number in connectionstrings.
        - "5434:1433"
        volumes:
        # Map [local directory:container directory] - this is so that db/log files are
        # stored on the "host" (your local computer, outside of container) and thereby 
        # persisted when container restarts.
        # by starting local path with "." it gets relative to current folder, meaning that the database
        # files will be on your computer in the same directory as you have this docker-compose.yaml file
        - ./data/mssql/data:/var/opt/mssql/data
        - ./data/mssql/log:/var/opt/mssql/log
        entrypoint: 
        # Due to an issue with the sqlserver image, permissions to db-files may be lost on container restart
        # by using the specific permissions_check entrypoint we assert that permissions are set on every restart
        - /bin/sh
        - -c
        - "/opt/mssql/bin/permissions_check.sh && /opt/mssql/bin/sqlservr"

    # direct-payment:
    #   image: registry.litium.cloud/apps/direct-payment:latest
    #   platform: linux/amd64
    #   dns: 
    #   - 192.168.65.2
    #   restart: unless-stopped
    #   ports:
    #   - "10010:80"
    #   - "10011:443"
    #   environment:
    #   # Enable HTTPS binding
    #   - ASPNETCORE_URLS=https://+;http://+
    #   - ASPNETCORE_HTTPS_PORT=10011
    #   # Configuration for HTTPS inside the container, exported dotnet dev-certs with corresponding password
    #   - ASPNETCORE_Kestrel__Certificates__Default__Password=SuperSecretPassword
    #   - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/localhost.pfx
    #   # Folder for the configuraiton, this is volume-mapped
    #   - CONFIG_PATH=/app_config
    #   # Folder where logfiles should be placed, this is volume-mapped
    #   - APP_LOG_PATH=/logs
    #   # Don't validate certificates
    #   - AppConfiguration__ValidateCertificate=false
    #   # Url to this app
    #   - AppMetadata__AppUrl=https://host.docker.internal:10011
    #   # Url to the litium installation
    #   - LitiumApi__ApiUrl=https://bookstore.localtest.me:5001
    #   volumes:
    #   - ./data/direct-payment/config:/app_config
    #   - ./data/direct-payment/data:/app_data
    #   - ./data/direct-payment/logs:/logs
    #   - ./data/direct-payment/DataProtection-Keys:/root/.aspnet/DataProtection-Keys
    #   - ./data/https:/https:ro

    # direct-shipment:
    #   image: registry.litium.cloud/apps/direct-shipment:latest
    #   platform: linux/amd64
    #   dns: 
    #   - 192.168.65.2
    #   restart: unless-stopped
    #   ports:
    #   - "10020:80"
    #   - "10021:443"
    #   environment:
    #   # Enable HTTPS binding
    #   - ASPNETCORE_URLS=https://+;http://+
    #   - ASPNETCORE_HTTPS_PORT=10021
    #   # Configuration for HTTPS inside the container, exported dotnet dev-certs with corresponding password
    #   - ASPNETCORE_Kestrel__Certificates__Default__Password=SuperSecretPassword
    #   - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/localhost.pfx
    #   # Folder for the configuraiton, this is volume-mapped
    #   - CONFIG_PATH=/app_config
    #   # Folder where logfiles should be placed, this is volume-mapped
    #   - APP_LOG_PATH=/logs
    #   # Don't validate certificates
    #   - AppConfiguration__ValidateCertificate=false
    #   # Url to this app
    #   - AppMetadata__AppUrl=https://host.docker.internal:10021
    #   # Url to the litium installation
    #   - LitiumApi__ApiUrl=https://bookstore.localtest.me:5001
    #   volumes:
    #   - ./data/direct-shipment/config:/app_config
    #   - ./data/direct-shipment/data:/app_data
    #   - ./data/direct-shipment/logs:/logs
    #   - ./data/direct-shipment/DataProtection-Keys:/root/.aspnet/DataProtection-Keys
    #   - ./data/https:/https:ro
        
    ```

1. Create a certificate file in the `<PROJECT_DIR>/.devsupport` folder with the following command

    ```bash
    dotnet dev-certs https -ep ./data/https/localhost.pfx -p SuperSecretPassword
    ```

1. Run **docker compose** to install the images from the `docker-compose-mac.yml` file

    ```bash
    docker compose -f <PROJECTR_DIR>/.devsupport/docker-compose-mac.yml up
    ```

    Make sure that the images runs correctly through the Docker Dashboard.

    Stop the process by hitting `ctrl+c`. You can then run the images daemonized from the Docker Dashboard instead.

1. If new project: Install Litium Accelerator

    ```bash
    # 1 - Install Litium Accelerator template
    dotnet new --install "Litium.Accelerator.Templates"

    # 2 - Steop into project folder
    cd <PROJECT_DIR>

    # 3 - Install a new Accelerator site using the Litium Accelerator template:
    dotnet new litmvcacc
    ```

1. If new project: Install project files

    ```bash
    # 1
    cd <PROJECT_DIR>/Src

    yarn install

    # 2
    cd Litium.Accelerator.Mvc

    yarn install
    ```

1. If new project: Add the SQL server ConnectionString

    Add the connection string to the database. Something like the string below

    ```json
    "Litium": {
        "Data": {
            "ConnectionString": "Server=<IP_ADDRESS>; Database=<DATABASE>; User Id=<USERID>; Password=<PASSWORD>;"
    }
    ```

1. If new project: Add project config in the file `<PROJECT_DIR>/Src/Litium.Accelerator.Mvc/Litium.Accelerator.Mvc.csproj` and add the following content just before the closing `</Project>` tag

    ```xml
        <Target Name="CreateDockerArguments" BeforeTargets="ContainerBuildAndLaunch">
            <PropertyGroup>
                <!-- Always pull to ensure that the latest image is used -->
                <DockerfileBuildArguments>--pull</DockerfileBuildArguments>
                
                <!-- Define the parameters for host folders -->
                <DockerLitiumFiles>$(MSBuildThisFileDirectory)../files</DockerLitiumFiles>
                <DockerLitiumLogs>$(DockerLitiumFiles)/logs</DockerLitiumLogs>
                
                <!-- Define volume mappings so that folders in the container are synced with 
                folders on host. The Docker image used (defined in the Dockerfile) already contains 
                the environment variable Litium__Folder__Local that defines files to 
                be stored in app_data inside the container -->
                <DockerfileRunArguments>$(DockerfileRunArguments) -v $(DockerLitiumFiles):/app_data:rw</DockerfileRunArguments>
                <DockerfileRunArguments>$(DockerfileRunArguments) -v $(DockerLitiumLogs):/app/bin/$(Configuration)/logs:rw</DockerfileRunArguments>

                <!-- Configure the container to use the dnsresolver-container as DNS: -->
                <DockerfileRunArguments>$(DockerfileRunArguments) --dns 192.168.65.2</DockerfileRunArguments>
            </PropertyGroup>
        </Target>
    </Project>
    ```

1. If new project: Create new database on the SQL Server started in the Docker container
    
    Easiest is to use the [DataGrip](https://www.jetbrains.com/datagrip/) database client.

    Connect to the server with the address and credentials that is set up in the `docker-compose.yml´ file.

    Create a new database with your preferred name same as the previous step for the connection string.

1. If new project: Migrate data to new database and create backoffice admin user

    ```bash
    cd <PROJECT_DIR>

    # Run database migrations and configure the database
    dotnet litium-db update --connection "Pooling=true;User Id=sa;Password=Pass@word;Database=<DATABASE>;Server=kubernetes.docker.internal,5434"

    # Create a new Litium backoffice admin user in the database with login admin/nimda
    dotnet litium-db user --connection "Pooling=true;User Id=sa;Password=Pass@word;Database=<DATABASE>;Server=kubernetes.docker.internal,5434" --login admin --password nimda
    ```

1. Build backoffice and frontend project

    ```bash
    # 1 - Backoffice
    cd <PROJECT_DIR>/Src

    yarn build

    # 2 - Frontend
    cd Litium.Accelerator.Mvc

    yarn build
    ```

    The frontend can be built with the watch flag during development.

    ```bash
    yarn build:w
    ```

1. Fire up the server with .NET 6

    ```bash
    dotnet run
    ```
