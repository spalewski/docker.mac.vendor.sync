# Improve Docker performance on Mac with vendor folder sync kept.

If you read this, I guess that you work on Mac, and you still looking for some smooth solution to work with docker on mac. So yeah, that’s the fact. Docker on Mac is slower than on Linux because of file system mechanism. And yeah, that’s why if you type query on google like “docker slow on mac” you will get a lot of results. Many of them are quite old and are fixed. But file system problem persists.

There is no better solution right now than keep as minimally amount of files and directories synced with host as we can. And that is also one of ideas of this nice docker setup from Mark Shust: <https://github.com/markshust/docker-magento>

So it works fast. It work’s nice. But what if you found some bug in vendor module and you want to quickly add some var\_dump() or exit() in some place? You must manually copy files from and to container after every line of code changed. It’s not comfortable for me in longer time. If we sync vendor with host we lose performance. If we have vendor only inside container, we lose full flexibility.

## So what we can do?

I found this problem in one my current projects. Vendor structure of this project is very complex. We have a lot of our extensions in vendor but also external modules. Magento with synced vendor was not usable.  So I though about some sftp. And PHPStorm remote deploy mechanism. 

I found this <https://hub.docker.com/r/emberstack/sftp> - ready to use container with sftp, easy configuration and ARM64 support (for Mac M1). And how to use that?

At yours docker-compose.yaml in part that have volumes add docker volume mapped with vendor folder inside container. 

version: "3"

services:
`  `app:
`    `image: markoshust/magento-nginx:1.18-8
`    `ports:
`      `- "80:8000"
`      `- "443:8443"
`    `volumes: &appvolumes
`      `- ~/.composer:/var/www/.composer:cached
`      `- ~/.ssh/id\_rsa:/var/www/.ssh/id\_rsa:cached
`      `- ~/.ssh/known\_hosts:/var/www/.ssh/known\_hosts:cached
`      `- appdata:/var/www/html
`      `- vendor:/var/www/html/vendor

After that, add emberstack/sftp as a service with added vendor volume:

sftp:
`  `image: "emberstack/sftp"
`  `ports:
`    `- "2222:22"
`  `volumes:
`    `- ./docker\_config/config/sftp.json:/app/config/sftp.json:ro
`    `- vendor:/home/docker/upload

as you can see, I have config file in: ./docker\_config/config/sftp.json

and it looks like that:

{
`  `*"Global"*: *{
`    `"Chroot"*: {
`      `*"Directory"*: *"%h"*,
`      `*"StartPath"*: *"/home/docker/upload"*
`    `},
`    `*"Directories"*: [*"upload"*]
`  `},
`  `*"Users"*: [
`    `{
`      `*"Username"*: *"docker"*,
`      `*"Password"*: *"docker"*
`    `}
`  `]
}

After all we have to define volumes with vendor added and driver:local. For ex.

volumes:
`  `appdata:
`  `dbdata:
`  `rabbitmqdata:
`  `sockdata:
`  `ssldata:
`  `vendor:
`    `driver: local

No we can do docker-compose up and and voila, we have sftp server with vendor directory. 

So let’s configure PHPStorm to use that.

1. Open PhpStorm->Preferencess->Build,Execution,Deployment and click + SFTP to add configuration:
1. Create SSH configuration -> click … and: user name: docker, password: docker, port 2222 . Test Connection.![](Aspose.Words.4520979d-0ecd-40c8-8737-f120418171c0.001.png)
1. Select newly created configuration: ![](Aspose.Words.4520979d-0ecd-40c8-8737-f120418171c0.002.png)
1. And go to Mappings and set like this: ![Obraz zawierający tekst

Opis wygenerowany automatycznie](Aspose.Words.4520979d-0ecd-40c8-8737-f120418171c0.003.png)
1. And under deployment options I recommend settings like this: ![Obraz zawierający tekst

Opis wygenerowany automatycznie](Aspose.Words.4520979d-0ecd-40c8-8737-f120418171c0.003.png)

And that’s it. Now when you go to directory structure and click right button on vendor you’ll see that option Deployment is active. It will also sync files after every save of file in vendor according to previous configuration:

![](Aspose.Words.4520979d-0ecd-40c8-8737-f120418171c0.004.png)

# Summarizing:
Docker on mac is still much slower than on linux. Best way to improve performance is to keep as many files as we can inside container without sync with host. With this solution you can still keep vendor files inside container synced with your host and have comfort of working with code in vendor without additional steps like copying from host to container, without any impact on performance.

Enjoy! 
