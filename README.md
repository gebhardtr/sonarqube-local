# SonarQube Local

## Getting started

### Install dependencies
```bash
brew install podman podman-compose
```

### Initialize podman
```bash
podman machine init
podman machine start
```

### Set required secrets

Create two new files:
* `sonar-secret` containing your desired SonarQube secret
* `postgres-secret` containing your desired PostgreSQL secret.

Create two new Podman secrets using the values from the files you created:
```bash
podman secret create sonar_jdbc_password /path/to/sonar-secret
podman secret create postgres_password /path/to/postgres-secret
```

### Run server
```bash
podman compose up --detach
```

The Compose file pins exact SonarQube and PostgreSQL versions. Do not replace
them with floating tags such as `community` or `latest`: a pull can otherwise
skip a required database-migration release.

### Generate repository scan token
1. Navigate to local SonarQube instance
```bash
open http://localhost:9000
```
1. Login using `admin:admin`
1. Click account icon in the upper right corner of the screen and select `My Accounts`
1. Click the `Security` tab
1. Generate token with:
    1. Name: global
    1. Type: Global Analysis Token
    1. Expires in: 30 days
1. Save token as a podman secret
    1. From a terminal, type the following but DO NOT press ENTER:
    ```bash
    pbpaste | podman secret create sonar_token --replace -
    ```
1. Copy the token from the SonarQube console in your browser by clicking the copy icon
1. Press RETURN in the terminal
1. Paste the token you copied into the terminal


### Scan repository
From within the repository directory:
```bash
podman run --rm -v $(pwd):/usr/src \
    --network sonarqube-local_default \
    --secret sonar_token,type=env,target=SONAR_TOKEN \
    sonarsource/sonar-scanner-cli \
    -Dsonar.projectKey=PLATFORM-repository -Dsonar.projectName='PROJECT-NAME' \
    -Dsonar.host.url=http://sonarqube:9000
```

## Stop the server
```bash
podman compose down
```

## Upgrade the server

Back up the database before changing either image version. If the current
Compose configuration still starts successfully, create a backup with:

```bash
podman compose up --detach db
podman compose exec -T db pg_dump -U sonar -d sonar -Fc > sonar-before-upgrade.dump
```

Check the [SonarQube update path](https://docs.sonarsource.com/sonarqube-community-build/server-update-and-maintenance/update/determine-path)
before changing the pinned SonarQube version. Calendar-year boundaries can
require an intermediate release. After starting each required version, open
`http://localhost:9000/setup` to complete its database migration before moving
to the next version.

PostgreSQL major versions require a dump and restore into a new data volume;
do not start a newer PostgreSQL image directly on a volume initialized by an
older major version.

### Upgrade an existing installation from PostgreSQL 12

The current Compose file uses a new PostgreSQL 17 volume so that it never opens
the PostgreSQL 12 data directory with an incompatible server. The old volume is
preserved during this procedure.

Stop the existing deployment, start PostgreSQL 12 temporarily against its old
volume, and create a dump. If your Compose project has a different name,
replace `sonarqube-local_postgresql_data` with the name shown by
`podman volume ls`.

```bash
podman compose down
podman run --detach --name sonarqube-postgres12-backup \
    --volume sonarqube-local_postgresql_data:/var/lib/postgresql/data \
    postgres:12
podman exec sonarqube-postgres12-backup pg_dump \
    --username sonar --dbname sonar --format custom > sonar-before-postgresql17.dump
podman stop sonarqube-postgres12-backup
podman rm sonarqube-postgres12-backup
```

Start PostgreSQL 17 and restore the dump before starting SonarQube:

```bash
podman compose up --detach db
podman compose exec -T db pg_restore --username sonar --dbname sonar \
    --clean --if-exists --no-owner --no-privileges \
    < sonar-before-postgresql17.dump
```

Migrate SonarQube through the required 26.1 bridge release:

```bash
SONARQUBE_IMAGE=sonarqube:26.1.0.118079-community \
    podman compose up --detach sonarqube
```

Open `http://localhost:9000/setup` and complete the migration. Wait until the
server is operational, then stop the bridge release and start the pinned target
release:

```bash
podman compose stop sonarqube
podman compose up --detach sonarqube
```

Open `http://localhost:9000/setup` once more to complete the final migration.
Keep `sonar-before-postgresql17.dump` and the old PostgreSQL 12 volume until the
upgraded instance and project data have been verified.
