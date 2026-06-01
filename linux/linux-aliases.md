```bash
# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

if [ -f ~/.bash_functions ]; then
    . ~/.bash_functions
fi
```

### version for RHEL-based distros (Amazon Linux, Rocky, AlmaLinux, CentOS, etc.

```bash
## Morning Opertures
alias whatsup='systemctl list-units --state=running'
alias hello='sudo systemctl stop httpd && cd workspace/project && ddev start && ddev launch'
alias hi='sudo systemctl stop httpd'
alias iad='systemctl is-active docker'
alias ports='nmap localhost'
alias dns="nmcli dev show | grep 'IP4.DNS'"
alias bye='shutdown -r now'

## Usual Instructions
alias yep='sudo dnf install $1'
alias nop='sudo dnf remove $1'
alias c='clear'
alias h='history'
alias hg='history | grep $1'
alias wg='wget -c '
alias al="echo '------------Your current aliases are:------------';alias"
alias sup="sudo dnf update -y"

## Content in folders
### Getting info from a position in a folder.

alias ll='ls -la --color=auto'
alias lf='ls -alF --color=auto'
alias la='ls -A --color=auto'
alias ls='ls -CF --color=auto'
alias lt='ls --human-readable --size -1 -S --classify --color=auto'
alias lh='ls -ahlt --color=auto'
alias lu='du -sh * | sort -h'
alias lc='find . -type f | wc -l'
alias ld='ls -d */ --color=auto'


## Files, folders and resources
alias fh='find . -name '
alias ..='cd ..'
alias ...='cd ../..'  
alias ....='cd ../../..'

### More Jump down
alias 1d="cd .."
alias 2d="cd ..;cd .."
alias 3d="cd ..;cd ..;cd .."
alias 4d="cd ..;cd ..;cd ..;cd .."
alias 5d="cd ..;cd ..;cd ..;cd ..;cd .."
alias untar='tar -zxvf $1'
alias tar='tar -czvf $1'
alias mnt="mount | awk -F' ' '{ printf \"%s\t%s\n\",\$1,\$3; }' | column -t | egrep ^/dev/ | sort"
alias df="df -Tha --total"
alias exp='nautilus .'  # swap for 'thunar .' or 'xdg-open .' depending on your desktop
alias std="stat -c '%y - %n' * | sort -r -t'-' -k1,1"

## Git Related Aliases
### Basic info
alias gs='git status'
alias gb='git branch'
alias gr='git remote -v'

### Getting info from 'Git log'
alias gl='git log --oneline'
alias glc="git log --format=format: --name-only --since=12.month | egrep -v '^$' | sort | uniq -c | sort -nr | head -50"
alias gld='git log --oneline --decorate --graph --all'
alias glp="git log -g --grep='PHP' -10 --pretty='%h - %s - %cn - %cd'"
alias glf='git for-each-ref --sort=-committerdate'

### Pushing to basic branches
alias gpom='git push origin master'
alias gpod='git push origin develop'

```

### ubuntu style
```bash
## Morning Opertures
alias whatsup='service --status-all'  
alias hello='sudo /etc/init.d/apache2 stop && cd workspace/project && ddev start && ddev launch'   
alias hi='sudo systemctl stop apache2'  
alias iad='systemctl is-active docker'  
alias ports='nmap localhost'
alias dns="sudo systemd-resolve --status | grep 'DNS Servers'"
alias bye='shutdown -r now'  

## Usual Instructions  
alias yep='sudo apt install $1'
alias nop='sudo apt remove $1'
alias c='clear'  
alias h='history'  
alias hg='history | grep $1'  
alias wg='wget -c '  
alias al="echo ------------Your curent aliases are:------------¡';alias"  
alias sup="sudo apt update && sudo apt upgrade -y"  

## Content in folders  
### Getting info from a position in a folder.
alias ll='ls -la --color=auto'
alias lf='ls -alF --color=auto'
alias la='ls -A --color=auto'
alias ls='ls -CF --color=auto'
alias lt='ls --human-readable --size -1 -S --classify --color=auto'
alias lh='ls -ahlt --color=auto'
alias lu='du -sh * | sort -h'
alias lc='find . -type f | wc -l'
alias ld='ls -d */ --color=auto'
    
## Files, folders and resources  
alias fh='find . -name '   
alias ..='cd ..'  
alias ...='cd ../..'  
alias ....='cd ../../..'
### More Jump down  
alias 1d="cd .."  
alias 2d="cd ..;cd .."  
alias 3d="cd ..;cd ..;cd .."  
alias 4d="cd ..;cd ..;cd ..;cd .."  
alias 5d="cd ..;cd ..;cd ..;cd ..;cd .."  
alias untar='tar -zxvf $1'  
alias tar='tar -czvf $1'  
alias mnt="mount | awk -F' ' '{ printf \"%s\t%s\n\",\$1,\$3; }' | column -t | egrep ^/dev/ | sort"  
alias df="df -Tha --total"   
alias exp='nautilus .'
alias std="stat -c '%y - %n' * | sort -r -t'-' -k1,1"
# Gets a list of files ordered by date.

## Git Related Aliases  
### Basic info
alias gs='git status'  
alias gb='git branch'  
alias gr='git remote -v'  

### Getting info from 'Git log'  
alias gl='git log --oneline'  
alias glc="git log --format=format: --name-only --since=12.month | egrep -v '^$' | sort | uniq -c  | sort -nr | head -50"  
alias gld='git log –oneline –decorate –graph –all'  
alias glp="git log -g --grep='PHP' -10 --pretty='%h - %s - %cn - %cd'"
alias glf='git for-each-ref --sort=-committerdate'   

### Pushing to basic branches 
alias gpom='git push origin master'  
alias gpod='git push origin develop'  

```