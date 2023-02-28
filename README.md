## Docker

Changes made:

- setup.py - orders  
  `change version -> alembic==1.9.3`  
  `add -> importlib-metadata==4.13.0`

- setup.py - products  
  `add -> importlib-metadata==4.13.0`

- setup.py - gateway  
  `add -> importlib-metadata==4.13.0`

Due the following errors:

https://discourse.jupyter.org/t/importerror-cannot-import-name-bindparamclause-from-sqlalchemy-sql-expression/17938
https://stackoverflow.com/questions/73929564/entrypoints-object-has-no-attribute-get-digital-ocean

### Reasons to add/update dependencies in setup.py file:

The conda installed different packages and specific versions for the application to work locally. But when I tried to deploy using docker, the errors occurred according to the links provided. So I decided to use the versions in the environment_dev.yml file, which worked in docker.

--------------------------------------------------------------------
## K8s

I followed the steps in the tutorial and only needed to make the following changes:

- Adding bitname chart repo  
  `-> the stable repository was deprecated so I used bitnami's to get the same services.`

- Change secret ref in orders app due bitnami postgres change the password key ref:

From:

            valueFrom:
              secretKeyRef:
                name: db-postgresql
                key: postgresql-password

To:

            valueFrom:
              secretKeyRef:
                name: db-postgresql
                key: postgres-password

- Add orders database on the new helm chart:

	`helm upgrade db bitnami/postgresql --install \
		--set global.postgresql.auth.database=orders`


--------------------------------------------------------------------


## Epinio

To deploy the application on Epinio, I used the method with container image.
For this I created a local repository via docker:

`container run -d --name registry.localhost -v local_registry:/var/lib/registry --restart always -p 5000:5000 registry:2`

I created a cluster using k3d using to the local repository:

`k3d cluster create epinio --registry-config "/home/erick/k3d-registries.yaml"`

`k3d-registries.yaml:`
``` yaml
mirrors:
  "registry.localhost:5000":
    endpoint:
      - http://registry.localhost:5000`
```

Then I connected the local registry to k3d's network

`docker network connect k3d-epinio registry.localhost`

After that, I followed the step-by-step procedure to install Epinio and all its dependencies.

The images that were pushed to the local registry will become available to Epinio.

Since the container images were already built locally, due to the previous steps, I just pushed them into the local registry, like this:

`
docker tag nameko/nameko-example-products:dev registry.localhost:5000/products:latest
docker push registry.localhost:5000/products:latest
`  
To upload the application, I used the GUI to get to know the tool, but you can also use:  
`epinio push --name Products --container-image-url registry.localhost:5000/products:latest`  
By adding `-env`, I added variables reference to database host, rabbitmq host, passwords.

Epinio offers postgres, redis and rabbitmq as a service, but I preferred to use helm charts as the previous steps.

## Why did I decide to use the container image?

I have not studied and read the in-depth documentation about nameko/epinio. When I tried to upload the python app using the normal code push, the application would not start, and this was because the `run.sh` was not executed.

So I decided to use the images that were already built instead of building new images, and use Epinio to provide the parameters like database, password, hosts.

It could be done in a better way, like creating an image with the variable export by epinio configuration. This way, it would not be necessary to add envs for each variable when creating the application, it would only be necessary to bind the desired epinio configuration.

## Problems with the container image

The applications run on port 8000.
However, when I deployed through epinio, traefik created the ingress and the ClusterIPs. But it created everything automatically using port 8080. As a result, the application did not work as expected and I had to change the ingress and the ClusterIP to work with port 8000. 
I didn't read all the epinio documentation, but I didn't find anything to configure the desired port when creating the application through epinio with the container image option. 