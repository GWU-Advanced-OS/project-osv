\newpage

# Thoughts on OSv

Here the authors provide their personal thoughts on the system.

## Sarah Morin

I thoroughly enjoyed learning about OSv. I enjoy the idea of systems attempting to compromise the pros and cons of virtual machines and containers; however, I'm not sure if it is anything more than an interesting idea. Throughout my research I found much more information specific to clean-slate unikernels than I found for OSv or legacy unikernels in general. It makes me beg the question: are the benefist of OSv and other legacy unikernels over containers and full VMs actually worth the effort? Are clean-slate unikernels the preferred unikernel? I don't have enough experience in production level cloud system to venture a guess the costs and benefits of clean-slate vs. legacy; however, the internet certainly seems more tantilized by the clean-slate option.

## Graham Schock
I have always enjoyed looking at Unikernels. OSv was written from a different perspective than other Unikernels I have looked at. Before this research project I looked into [MirageOS](https://github.com/mirage/mirage) which had a hands on build and configuration process where the user had to write OCaml in order to get there applications fused into their Unikernel. It was difficult even getting the OCaml compiler and dependencies to work. It was even harder to learn and understand OCaml enough to configure MirageOS with a simple `hello world` application. However, OSv did not have this problem. I was easily able to create my own applications and run them immedietly with OSv. I think this is the major upside to OSv, but can also be a downside. I think we lost the true Unikernel approach and get a hybrid DOS kernel. Whether I compiled mySQL or hello_world.c into my Unikernel they were both almost the same size but had very different requirments. However, I think the ease of use in OSv takes Unikernels in the right direction in where they need to go. 
