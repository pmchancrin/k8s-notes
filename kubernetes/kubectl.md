# kubectl bash completion

> https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion 

## install and configure
on centos

```
$ yum install -y bash-completion
$ source <(kubectl completion bash) 
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
```

for more details, consult *==kubectl completion -h==* 
## issue

```
$ kubectl bash: _get_comp_words_by_ref: command not found
```

**solution**

```
$ source /etc/bash_completion
or
$ source /etc/profile.d/bash_completion.sh
and then
$ source <(kubectl completion bash)
```

# usage