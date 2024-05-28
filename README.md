```
List your tools here:
AWS RDS, ECR, ECS, CircleCi, Postgresql, Docker 
```

<img width="1095" alt="Screenshot 2024-05-28 at 10 30 26" src="https://github.com/sukarnozhang/devops_project/assets/78150905/794cf95e-3793-43e8-a9dc-6813475509d3">

```yml
# your final config.yml file content here at the end of the project
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0
  aws-ecr: circleci/aws-ecr@9.0
  aws-ecs: circleci/aws-ecs@0.0.8
  docker: circleci/docker@2.4.0
  snyk: snyk/snyk@1.5.0

jobs:
  build:
    description: "build docker"
    docker:
      - image: cimg/openjdk:19.0.1
    steps:
      - checkout
      - run:
          name: Build
          command: mvn -B -DskipTests clean package
      - persist_to_workspace:
          root: .
          paths:
            - .
  test:
    description: "test springboot"
    docker:
      - image: cimg/openjdk:19.0.1
    steps:
      - checkout
      - run:
          name: Build
          command: mvn test

  scan:
    docker:
      - image: cimg/openjdk:19.0.1
    environment:
      IMAGE_NAME: devops
    steps:
      - checkout
      - setup_remote_docker
      - docker/check
      - run: docker build -t $IMAGE_NAME .
      - snyk/scan:
          docker-image-name: $IMAGE_NAME

workflows:
  simple_workflow:
    jobs:
      - build
      - test:
          requires:
            - build
      - scan:
          requires:
            - test 
      - aws-ecr/build_and_push_image:
          account_id: ${AWS_ACCOUNT_ID}
          auth:
            - aws-cli/setup:
                role_arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/oidc-ecr
                role_session_name: mysession@${ORGANIZATION_ID}
          dockerfile: Dockerfile
          repo: devops
          tag: latest
          filters:
            branches:
              only:
                - release
          requires:
            - test
      - aws-ecs/deploy-service-update:
          cluster-name: test
          family: test   ### name of the task definition (image URL)
          service-name: test
          requires:
            - aws-ecr/build_and_push_image

```

END
