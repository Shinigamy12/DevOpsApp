**commande docker database:** docker run --network app-network --name database Bastien/database
    docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
    
    
    
    

## Question
### 1-1 Why use `-e` at runtime instead of putting environment variables in the Dockerfile?

It is better to pass environment variables with `-e` when running the container because:

- **Security:** sensitive values (passwords, API keys, tokens) are not hardcoded in the Dockerfile. Anything written in the Dockerfile with `ENV` is stored in the image layers and can be read by anyone who has the image (`docker history`, `docker inspect`), or by anyone who can see the Dockerfile in the Git repository.
- **Flexibility / reusability:** the same image can be used in different environments (dev, test, prod) with different values (database name, user, password) without rebuilding the image.
- **Separation of configuration and code:** the image contains the application, and the configuration is given at runtime. This follows the "12-factor app" principle.

