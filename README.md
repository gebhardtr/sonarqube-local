# SonarQube Local

## Getting started

### Install dependencies
```bash
brew install podman podman-compose
```

### Run server
```bash
podman compose up --detach
```

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
    -Dsonar.projectKey=PLATFORM-respository -Dsonar.projectName='PLATFORM-repository' \
    -Dsonar.host.url=http://sonarqube:9000
```

## Stop the server
```bash
podman compose down
```
