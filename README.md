# Oh Es Root
![Banner](logo.png)
This is the main repo for the oh es operating system. This is mostly a container for a bunch of submodules.i
## Developmnet
The kernel uses GNU Make and Cargo as its build system. It requires nightly rust, nasm, llvm and GNU ld cross compiled for x86-64 and libgcc cross compiled for x86-64 at /opt/cross.
## Userland
The userland is built with a custom build system, ohbuild. It requires rust (the exact same nightly as kernel is tested) and GNU ld cross compiled for x86-64. Installing is done by navigating into ohbuild submodule and executing `cargo install --path .`. Then, ohbuild can be invoked by using `ohbuild --path path/to/cache/dir --out path/to/out/dir`.
## Updating submodules
There is a handy oneliner: 
```
git submodule update --remote kernel ohbuild && git add . && git commit -s -m "Update submodules" && git push origin main
```
## Generating code
Use `csa`. The name is _borrowed_ from v8's builtin codegen thing, CodeStubAssembler. You need `jq` for that though.
