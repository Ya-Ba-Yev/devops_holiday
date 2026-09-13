FROM jenkins-course-local:2.568.2-jdk21

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends ansible openssh-client \
    && ansible-galaxy collection install -p /usr/share/ansible/collections community.docker \
    && rm -rf /var/lib/apt/lists/*