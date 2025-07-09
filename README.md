# CircleCI configuration file for CI/CD pipeline
# Version specifies the CircleCI config version being used
version: 2.1

# Orbs are reusable packages of CircleCI config to simplify jobs
orbs:
  aws-cli: circleci/aws-cli@4.0       # AWS CLI commands support
  aws-ecr: circleci/aws-ecr@9.0       # AWS ECR  operations
  aws-ecs: circleci/aws-ecs@0.0.8     # AWS ECS  deployments
  docker: circleci/docker@2.4.0        # Docker commands support
  snyk: snyk/snyk@1.5.0               # Snyk security vulnerability scanning

jobs:
  build:
    description: "Build Docker image using Maven"
    docker:
      - image: cimg/openjdk:19.0.1     # Docker image with OpenJDK 19 for Java build
    steps:
      - checkout                      # Checkout source code from repo to circleci workspace
      - run:
          name: Build
          command: mvn -B -DskipTests clean package  # Build project, skipping tests, there is security scan and mavern unit test
      - persist_to_workspace:         # Save workspace for subsequent jobs
          root: .
          paths:
            - .

  test:
    description: "Run tests for Spring Boot application"
    docker:
      - image: cimg/openjdk:19.0.1
    steps:
      - checkout
      - run:
          name: Run Tests
          command: mvn test             # Execute unit and integration tests

  scan:
    description: "Run security scan on Docker image using Snyk"
    docker:
      - image: cimg/openjdk:19.0.1
    environment:
      IMAGE_NAME: devops               # Define Docker image name for scanning
    steps:
      - checkout
      - setup_remote_docker            # Enable Docker daemon for building images
      - docker/check                  # Verify Docker environment is set
      - run: docker build -t $IMAGE_NAME .  # Build Docker image locally
      - snyk/scan:
          docker-image-name: $IMAGE_NAME    # Scan Docker image for vulnerabilities

workflows:
  simple_workflow:
    jobs:
      - build                        # Build job runs first
      - test:
          requires:
            - build                 # Test job depends on successful build
      - scan:
          requires:
            - test                  # Scan job depends on successful test
      - aws-ecr/build_and_push_image:  # Push built image to AWS ECR
          account_id: ${AWS_ACCOUNT_ID}
          auth:
            - aws-cli/setup:
                role_arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/oidc-ecr  # IAM role for ECR access
                role_session_name: mysession@${ORGANIZATION_ID}
          dockerfile: Dockerfile
          repo: devops
          tag: latest
          filters:
            branches:
              only:
                - release            # Run only on release branch
          requires:
            - test
      - aws-ecs/deploy-service-update:  # Deploy updated image to ECS service
          cluster-name: test
          family: test                # ECS task definition family name (usually image URL)
          service-name: test         # ECS service to update
          requires:
            - aws-ecr/build_and_push_image  # Deployment requires image to be pushed
