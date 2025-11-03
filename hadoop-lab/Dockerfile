FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    openjdk-11-jdk curl ssh rsync python3 ca-certificates procps nano sudo openssh-server \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -ms /bin/bash hadoop && echo "hadoop ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/hadoop
USER hadoop
WORKDIR /home/hadoop

# Hadoop install
ENV HADOOP_VERSION=3.3.6
RUN curl -L -o hadoop.tgz https://downloads.apache.org/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz \
    && tar -xzf hadoop.tgz && mv hadoop-${HADOOP_VERSION} hadoop && rm hadoop.tgz

# ENV VARS
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
ENV HADOOP_HOME=/home/hadoop/hadoop
ENV HADOOP_COMMON_HOME=$HADOOP_HOME
ENV HADOOP_HDFS_HOME=$HADOOP_HOME
ENV HADOOP_MAPRED_HOME=$HADOOP_HOME
ENV HADOOP_CONF_DIR=/home/hadoop/hadoop/etc/hadoop
ENV YARN_HOME=$HADOOP_HOME
ENV PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

# SSH
RUN mkdir -p /home/hadoop/.ssh && ssh-keygen -q -N "" -f /home/hadoop/.ssh/id_rsa \
    && cat /home/hadoop/.ssh/id_rsa.pub >> /home/hadoop/.ssh/authorized_keys

CMD ["/bin/bash"]

