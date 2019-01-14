# 详细命令说明

* https://docs.docker.com/engine/reference/builder/  
* https://docs.docker.com/develop/develop-images/dockerfile_best-practices/

# issue

## COPY和ADD区别
* https://stackoverflow.com/questions/24958140/what-is-the-difference-between-the-copy-and-add-commands-in-a-dockerfile  
* https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#add-or-copy


1. ADD 允许src是URL，但是强烈不建议，因为可能会导致image很大，建议使用wget或者curl，可以用完就删掉
2. 如果ADD的src是archive，tar包的话，会自动解包
3. 官网建议使用COPY，因为比较直观，ADD