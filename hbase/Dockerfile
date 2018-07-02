FROM centos:7
ENV JAVA_HOME /usr/lib/jvm/jre-1.8.0-openjdk
COPY hdp.repo /etc/yum.repos.d/
RUN yum updateinfo \
    && yum install -y java-1.8.0-openjdk-devel hbase phoenix hadoop-yarn hadoop-mapreduce \
    && yum clean all
