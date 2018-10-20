# libssh-0.5.0 legacy implementation
FROM ubuntu:trusty
ADD libssh-0.5.0.tar.gz /root
RUN apt-get update --assume-yes &&\
apt-get upgrade --assume-yes &&\
apt-get install vim zlib1g zlib1g-dev libssl-dev openssh-client cmake build-essential --assume-yes &&\
ssh-keygen -t dsa -f /root/ssh_host_dsa_key -N '' &&\
ssh-keygen -t rsa -b 2048 -f /root/ssh_host_rsa_key -N '' &&\
cd /root/libssh-0.5.0 &&\
mkdir build &&\
cd build &&\
cmake .. &&\
make 
EXPOSE 22
CMD ["/root/libssh-0.5.0/build/examples/samplesshd", "-d", "/root/ssh_host_dsa_key", "-k", "/root/ssh_host_rsa_key", "-p", "22", "-v", "0.0.0.0" ]
