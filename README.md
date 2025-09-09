# CircleCI Cheat Sheet

## Configuration (.circleci/config.yml)
```yaml
version: 2.1

orbs:
  node: circleci/node@5.0.3
  docker: circleci/docker@2.2.0
  aws-ecr: circleci/aws-ecr@8.2.1

executors:
  node-executor:
    docker:
      - image: cimg/node:18.17
    working_directory: ~/project
    resource_class: medium

commands:
  install-deps:
    steps:
      - restore_cache:
          keys:
            - deps-{{ checksum "package-lock.json" }}
            - deps-
      - run: npm ci
      - save_cache:
          key: deps-{{ checksum "package-lock.json" }}
          paths:
            - node_modules

jobs:
  test:
    executor: node-executor
    steps:
      - checkout
      - install-deps
      - run:
          name: Run tests
          command: npm test
      - store_test_results:
          path: test-results
      - store_artifacts:
          path: coverage

  build:
    executor: node-executor
    steps:
      - checkout
      - install-deps
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist

  deploy:
    docker:
      - image: cimg/aws:2023.09
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy to S3
          command: |
            aws s3 sync dist/ s3://$S3_BUCKET_NAME --delete
            aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_ID --paths "/*"

workflows:
  main:
    jobs:
      - test:
          filters:
            branches:
              ignore: main
      - build:
          requires:
            - test
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: main
          context: production
```

## Advanced Features
```yaml
# Matrix builds
jobs:
  test-matrix:
    parameters:
      node-version:
        type: string
      os:
        type: string
    docker:
      - image: cimg/node:<< parameters.node-version >>
    steps:
      - checkout
      - run: npm test

workflows:
  test-all:
    jobs:
      - test-matrix:
          matrix:
            parameters:
              node-version: ["16.20", "18.17", "20.5"]
              os: ["linux", "windows"]

# Conditional workflows
workflows:
  deploy-production:
    when:
      and:
        - equal: [ main, << pipeline.git.branch >> ]
        - not: << pipeline.parameters.skip-deploy >>
    jobs:
      - build
      - deploy
```

## Official Links
- [CircleCI Documentation](https://circleci.com/docs/)
- [Configuration Reference](https://circleci.com/docs/configuration-reference/)
- [Orbs Registry](https://circleci.com/developer/orbs)