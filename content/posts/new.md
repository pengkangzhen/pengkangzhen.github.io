---
title: "Poetry 开发依赖与 mypy 类型检查"
date: 2026-04-29
categories: ["技术"]
tags: ["Python", "Poetry", "mypy"]
summary: "介绍如何通过 pyproject.toml 管理开发依赖，并使用 mypy 进行静态类型检查。"
draft: false
---

## 1. 添加开发依赖到 pyproject.toml                                                                                                                                                          
                                                                                                                                                                                        
作用：安装用于类型检查和文档生成的工具                                                                                                                                                    
                                                                                                                                                                                        
添加的工具：                                                                                                                                                                              
- mypy: Python 静态类型检查器，用于在运行前发现类型错误                                                                                                                                   
- sphinx: Python 文档生成工具                                                                                                                                                             
- sphinx-rtd-theme: ReadTheDocs 风格的 Sphinx 文档主题                                                                                                                                    
- autodocsumm: Sphinx 扩展，用于自动生成 API 文档摘要                                                                                                                                     
                                                                                                                                                                                        
这些依赖被添加到 [tool.poetry.group.dev.dependencies] 部分，意味着它们只在开发时需要，不会影响生产环境的部署。                                                                            
                                                                                                                                                                                        
2. 通过 mypy 类型检查                                                                                                                                                                     
                                                                                                                                                                                        
作用：确保代码的类型安全性和正确性                                                                                                                                                        
                                                                                                                                                                                        
具体含义：                                                                                                                                                                                
- 所有 4 个核心 Agent 文件现在都有完整的类型注解（如 Dict[str, Any], Optional[int] 等）                                                                                                   
- mypy 工具验证了这些类型注解，发现 0 个类型错误                                                                                                                                          
- 这意味着：                                                                                                                                                                              
- 函数参数类型正确匹配                                                                                                                                                                  
- 返回值类型正确声明                                                                                                                                                                    
- 不会出现运行时的类型相关错误                                                                                                                                                          
                                                                                                                                                                                        
示例：                                                                                                                                                                                    
```
# 之前（没有类型注解）                                                                                                                                                                    
def forward_step(self, request):                                                                                                                                                          
                                                                                                                                                                                
                                                                                                                                                                                        
# 之后（有完整类型注解）                                                                                                                                                                  
def forward_step(self, request: Dict[str, Any]) -> Dict[str, Any]:                                                                                                                        
```                                                                                                                                                                                
                                                                                                                                                                                        
这提升了代码的可维护性和IDE支持（如自动补全、类型提示）。  