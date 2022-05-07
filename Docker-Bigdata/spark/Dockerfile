FROM bitnami/spark:latest
USER root
#**************************************** INSTALLLING java ******************************
RUN apt-get -y update
RUN apt-get -y upgrade 
RUN apt-get -y install rsync
RUN apt-get -y install vim sudo
RUN apt-get -y install telnet
RUN apt-get -y install locate
RUN apt-get -y install openssh-server
RUN apt-get -y install ssh
RUN ssh-keygen -t rsa -P '' -f /root/.ssh/id_rsa
RUN cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
RUN chmod 0600 /root/.ssh/authorized_keys
ADD ./config/ssh_config /etc/ssh/
EXPOSE 22
EXPOSE 8889
RUN echo "root:root" | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

#./jupyter notebook --ip 0.0.0.0 --allow-root
export PYSPARK_DRIVER_PYTHON=/anaconda3/bin/jupyter
export PYSPARK_DRIVER_PYTHON_OPTS="notebook --ip 0.0.0.0 --allow-root"
#PYTHONPATH=/opt/bitnami/spark/python/:
export PYSPARK_PYTHON=/anaconda3/bin/python
#https://medium.com/sicara/get-started-pyspark-jupyter-guide-tutorial-ae2fe84f594f#:~:text=There%20are%20two%20ways%20to,load%20PySpark%20using%20findSpark%20package