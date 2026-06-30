FROM rundeck/rundeck:5.20.1
USER root
RUN apt-get update -qq && \
    apt-get install -y -qq ansible sshpass && \
    ansible-galaxy collection install ansible.windows && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER rundeck
