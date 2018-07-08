# Reading Papers

## Operating Systems

### Unikernel

- [Jitsu: Just-In-Time Summoning of Unikernels](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-madhavapeddy.pdf)

## Memory Management

- Ingens
- HeteroOS
- SmartMD

- Clock-pro
	- [Clock-pro outperforms the clock replacement and has been adopted in OS kernels](http://web.cse.ohio-state.edu/~zhang.574/clockpro-usenix-05.html)
	- [A CLOCK-Pro page replacement implementation](https://lwn.net/Articles/147879/)
	- [Rik van Riel for clock-pro: Measuring Resource Demand on Linux](https://www.kernel.org/doc/ols/2006/ols2006v2-pages-295-302.pdf)

## Architecture and uArch

- [Optimizing the TLB Shootdown Algorithm with Page Access Tracking](https://www.usenix.org/system/files/conference/atc17/atc17-amit.pdf)

>The actions of one processor causing the TLBs to be flushed
>on other processors is what is called a TLB shootdown.
>
>A quick example:
>1. You have some memory shared by all of the processors in your system.
>2. One of your processors restricts access to a page of that shared memory.
>3. Now, all of the processors have to flush their TLBs, so that the ones that
>   were allowed to access that page can't do so any more.

- [UNified Instruction/Translation/Data (UNITD) Coherence](http://people.ee.duke.edu/~sorin/papers/hpca10_unitd.pdf)

- [Avoiding TLB Shootdowns Through Self-Invalidating TLB Entries](http://drona.csa.iisc.ac.in/~arkapravab/papers/pact2017_final_version.pdf)

## GPU Architecure

- [Parallel programming Introduction to GPU architecture](http://www.irisa.fr/alf/downloads/collange/cours/gpuprog_ufmg/gpuprog_1.pdf)

- [Introduction to GPU Architecture](http://haifux.org/lectures/267/Introduction-to-GPUs.pdf)

- [How a GPU Works](https://www.cs.cmu.edu/afs/cs/academic/class/15462-f11/www/lec_slides/lec19.pdf)

- [GPU Architectures A CPU Perspective](https://courses.cs.washington.edu/courses/cse471/13sp/lectures/GPUsStudents.pdf)

- [Graphics and Computing GPUs](https://pdfs.semanticscholar.org/386b/eedaffaeb23a5934f52b3821dd3b44183c77.pdf)

## Cloud & Virtualization

- [virtio-user]

## Security

- [Lock-in-pop]
- [kaiser]

