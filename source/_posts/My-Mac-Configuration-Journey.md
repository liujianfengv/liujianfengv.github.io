---
title: My Mac Configuration Journey
date: 2024-09-28 13:48:21
tags:
---
## Switch Caps Lock and Ctrl                                                                                                                                                                                          
                                                                                                                                                                                                                      
Systems Settings -> Keyboard -> Keyboard Shortcuts -> Modifier Keys -> Change Caps Lock key to Control -> Change Control key to Caps Lock key                                                                         
                                                                                                                                                                                                                      
## ZSH                                                                                                                                                                                                                
                                                                                                                                                                                                                      
```                                                                                                                                                                                                                   
#Git command Alias                                                                                                                                                                                                    
alias gst='git status'                                                                                                                                                                                                
alias gp='git push'                                                                                                                                                                                                   
alias gc='git commit'                                                                                                                                                                                                 
alias gb='git branch'                                                                                                                                                                                                 
alias gco='git checkout'                                                                                                                                                                                              
                                                                                                                                                                                                                      
#Show Git branch information on terminal                                                                                                                                                                              
COLOR_DEF=$'%f'                                                                                                                                                                                                       
COLOR_USR=$'%F{243}'                                                                                                                                                                                                  
COLOR_DIR=$'%F{197}'                                                                                                                                                                                                  
COLOR_GIT=$'%F{39}'                                                                                                                                                                                                   
setopt PROMPT_SUBST                                                                                                                                                                                                   
export PROMPT='${COLOR_USR}%n ${COLOR_DIR}%~ ${COLOR_GIT}$(parse_git_branch)${COLOR_DEF} $ '                                                                                                                          
                                                                                                                                                                                                                      
```

