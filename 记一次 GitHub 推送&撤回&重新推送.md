# 记一次 GitHub 推送&撤回&重新推送                                                                                                                                            
                                                                                                                                                                            
## 问题描述                                                                                                                                                                   
                                                                                                                                                                            
在推送代码时，不小心将 IDE 配置文件（`.idea/` 目录）一起推送到了远程仓库。实际上只需要提交业务相关的 `.tf` 文件。                                                             
                                                                                                                                                                            
## 问题现状                                                                                                                                                                   
                                                                                                                                                                            
远程分支包含了不应该提交的文件：                                                                                                                                              
- `.idea/.gitignore`                                                                                                                                                          
- `.idea/vcs.xml`                                                                                                                                                             
- `msf.tf`（需要保留）                                                                                                                                                        
                                                                                                                                                                            
## 解决步骤                                                                                                                                                                   
                                                                                                                                                                            
### 1. 检查当前提交状态                                                                                                                                                       
                                                                                                                                                                            
```bash                                                                                                                                                                       
git diff --name-status origin/main...HEAD                                                                                                                                     
                                                                                                                                                                            
输出显示当前分支比 main 多了三个文件，其中 .idea/ 相关文件需要移除。                                                                                                          
                                                                                                                                                                            
2. 撤销最后一次提交                                                                                                                                                           
                                                                                                                                                                            
使用 --soft 参数撤销提交，但保留文件改动：                                                                                                                                    
                                                                                                                                                                            
git reset --soft HEAD~1                                                                                                                                                       
                                                                                                                                                                            
此时所有改动回到暂存区（staged）状态。                                                                                                                                        
                                                                                                                                                                            
3. 重新选择需要提交的文件                                                                                                                                                     
                                                                                                                                                                            
取消不需要的文件暂存，只保留业务文件：                                                                                                                                        
                                                                                                                                                                            
git restore --staged .idea                                                                                                                                                    
git add msf.tf                                                                                                                                                                
                                                                                                                                                                            
4. 检查暂存状态                                                                                                                                                               
                                                                                                                                                                            
git status                                                                                                                                                                    
                                                                                                                                                                            
确认只有 msf.tf 在暂存区。                                                                                                                                                    
                                                                                                                                                                            
5. 重新提交并强制推送                                                                                                                                                         
                                                                                                                                                                            
git commit -m "add msf role"                                                                                                                                                  
git push -f                                                                                                                                                                   
                                                                                                                                                                            
注意：由于远程分支已经有旧的提交记录，需要使用 -f 参数强制推送来覆盖。                                                                                                        
                                                                                                                                                                            
关键命令说明                                                                                                                                                                  
                                                                                                                                                                            
- git reset --soft HEAD~1：撤销最近一次提交，改动保留在暂存区                                                                                                                 
- git restore --staged <file>：取消文件的暂存状态                                                                                                                             
- git push -f：强制推送，覆盖远程分支历史                                                                                                                                     
                                                                                                                                                                            
预防措施                                                                                                                                                                      
                                                                                                                                                                            
1. 推送前使用 git status 检查将要提交的文件                                                                                                                                   
2. 配置 .gitignore 文件，排除不需要版本控制的文件：                                                                                                                           
- IDE 配置目录（如 .idea/）                                                                                                                                                 
- 临时文件目录（如 .terraform/）                                                                                                                                            
- 系统文件（如 .DS_Store）                                                                                                                                                  
3. 使用 git diff --staged 查看暂存的具体改动                                                                                                                                  
                                                                                                                                                                            
总结                                                                                                                                                                          
                                                                                                                                                                            
通过 reset --soft 和 restore --staged 组合，可以灵活地调整提交内容，而不丢失已有改动。强制推送需谨慎使用，确保不会影响团队其他成员的工作。                                    
                                                                                                                                                                            
这份文档记录了完整的操作流程和注意事项，可以作为日后参考
