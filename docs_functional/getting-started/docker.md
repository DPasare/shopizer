# Run Shopizer with Docker Compose​

_Source: https://shopizer-ecommerce.github.io/documentation/docker.html_

# Run Shopizer with Docker Compose ​

![Docker](https://shopizer-ecommerce.github.io/documentation/assets/docker.fec6f20c.jpg)

Shopizer can be run from Docker containers. The following instructions will run Shopizer Headless, Shopizer Shop App, Shopizer Admin App and Mysql

sh
    
    
    git clone git@github.com:shopizer-ecommerce/shopizer-docker-compose.git
    cd shopizer docker compose

Now run Docker containers

sh
    
    
    docker compose up -d

Open a browser on this urls

Application| Url  
---|---  
Headless| localhost:8080  
Shop| localhost:3000  
Admin| localhost:4200  
  
Admin app credentials

Username: **[admin@shopizer.com](https://shopizer-ecommerce.github.io/documentation/<mailto:admin@shopizer.com>)**

Password: **password**
