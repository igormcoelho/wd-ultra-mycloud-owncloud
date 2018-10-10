FROM resin/rpi-raspbian:latest

#RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" >> /etc/apt/sources.list

RUN apt-get -y update 

RUN dpkg-divert --local --rename --add /sbin/initctl
#RUN ln -s /bin/true /sbin/initctl

RUN apt-get install -y locales dialog
RUN locale-gen en_US en_US.UTF-8
RUN dpkg-reconfigure -f noninteractive locales

RUN apt-get update && apt-get -y install cron dnsutils wget

ADD crontab /etc/cron.d/noip-cron

RUN mkdir /noipupdater
#RUN mkdir /noipupdater/configdir
#RUN mkdir /noipupdater/logdir

ADD ./noipupdater.sh /noipupdater/noipupdater.sh

RUN chmod 0644 /etc/cron.d/noip-cron

RUN crontab /etc/cron.d/noip-cron

RUN touch /var/log/cron.log

CMD ["cron", "-f"]

