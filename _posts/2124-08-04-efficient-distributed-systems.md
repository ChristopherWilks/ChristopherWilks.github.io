## Efficient Distributed Systems for Small-Medium Sized Jobs

* Start simple, only use more complex approach if absolutely required (e.g. linear vs. non-linear models)
* Opportunities to (easily) leverage parallelism are everywhere*, use them!
* Utilize a shared-nothing approach as much as possible when leveraging parallelism at the CPU, System, and Filesystem levels
* Compression (for cost savings) + indexing can dramatically speed up data access, even using S3

![image](https://github.com/user-attachments/assets/cd397b7c-37a2-4daf-9ce0-ee9c9e76e8fe)

source: https://mindmajix.com/snowflake-architecture

### Filesystems

* Critical but fragile (FSx [LUSTRE], NFS), especially in a distributed context
* Even fast distributed filesystems are not made to support highly concurrent, small-file IO
* Locality aware: /dev/shm > local SSDs > FSx > EBS SSDs > S3
* Independent, local SSDs (ephemeral disks) are available on the *d.#xlarge lines of instance types (e.g. C5d.4xlarge, R5d.4xlarge)
* Ephemeral filesystems (and /dev/shm) disappear on *stopping* the instance (not just terminating it)

### Shared-Nothing Distributed Filesystem Pattern

* Deploy instances with ephemeral, local SSDs or large memory size
* Create FS(s) on local SSD(s) and/or just use /dev/shm
* Download reference data in parallel chunks from S3 to local FS(s) from Step 2 on each VM separately
* Spread compute intensive, independent operations across data on local FS(s)
* When finished, upload results to S3 (in parallel), erase local data
* This pattern uses the speed of the networking/locally attached storage of *each* VM independently and is thus appropriate for massive scaling

![image](https://github.com/user-attachments/assets/21be16dd-f126-4dc7-a83b-1dddea22a886)

source: https://paris-cluster-2019.gitlabpages.inria.fr/cleps/cleps-userguide/going_further/lustre.html

