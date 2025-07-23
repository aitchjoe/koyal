---
title: Buildpacks TLDR
---

1. Buildpacks 用于简化从源码到生成容器镜像的中间过程，不需要操心如何配置 Dockerfile 却能达到很专业的 Dockerfile 效果。
1. Buildpacks 取代 Dockerfile 的工作原理并不是自动生成 Dockerfile，当然也不是写另一种形式的 Dockerfile，而是通过分析源码确定应用的技术栈、执行相应构建工作、并最终直接生成 [OCI](https://opencontainers.org/) 镜像。
1. Buildpacks 涵盖主流的开发语言和技术栈，因此以上"执行相应构建工作"是通过模块化、插件的形式来扩展实现的。
1. Buildpacks 简化的不只是 Dockerfile 这个领域，"执行相应构建工作"更多包含的是 CI 领域的任务，且更自动化更专业。
1. Buildpacks 不止标准化了构建过程，所生成的镜像本身比如目录结构等也具备一致的规范。
1. 但 Buildpacks 并不能完全取代 Dockerfile，而只是在"从源码到镜像"的场景，也未覆盖所有源码种类；**在限定场景的前提下 Buildpacks 的重点是简化和标准化**。
1. Buildpacks 的成熟度在 CNCF 仍是孵化器阶段（2025-06），但考虑到生态圈规模等因素，可以尝试预研了。
1. 从实操感受，目前（2023-10）Buildpacks 对企业环境（主要是 Restricted internet access / Wall）、OpenShift 的支持仍有欠缺，有些是非技术因素，但仍然说明了不足够成熟。
1. 由于成熟度及复杂性，当前（2025-06）更建议用户采用的是 [*Jib*](why-here.md) 方案，虽然 Jib 主要针对 Java 应用，但我们也尝试了前端 Nginx 镜像的 Jib 构建，效果还不错。
1. 强烈建议 DevOps 平台方跟进 Jib 或 Buildpacks 方案，但 DevOps 用户暂时只用考虑 Jib 方案。
1. Buildpacks 不会对标 Helm 等模板方案，取代它们的应该是 [*OAM*](why-here.md) 之类（研究中）。
1. 简化或者说傻瓜化并不代表真的傻瓜都能做，用户仍然需要学习基础的 Linux、[Container](container-tldr.md)、[Kubernetes](k8s-tldr.md)、[CI / CD](cicd-tldr.md) 知识和理念。
