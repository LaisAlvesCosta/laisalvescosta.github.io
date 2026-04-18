---
categories: [Dev, Kernel_Dev]
date: 2026-04-15
tags: [kernel, mac5856]
title: Linux Kernel Patch 1 Deduplications with 100% similarity
---

## Setting Up the Environment
As part of this course, we contributed to the Linux Kernel, a major free software project. From a list of potential tasks, my partner and I chose to focus on code duplication with 100% similarity. Among the suggestions, we chose the following two identical functions within the `drivers/gpu/drm/amd/amdgpu/` directory: 
- `gfx_v11_0.c :: gfx_v11_0_eop_irq`
- `gfx_v12_0.c :: gfx_v12_0_eop_irq` 
- Total length: 52 lines of identical code.
  
Since we are working on curated issues within the AMD DRM subsystem, we used the following commands to clone and update the repository:

```bash
cd linux-repositorio
git clone https://gitlab.freedesktop.org/agd5f/linux.git --branch amd-staging-drm-next
git pull

```

To resolve this duplication, our goal was to create a **helper function** in a common location, allowing both drivers to call it and eliminating redundancy. We searched for a suitable location and chose `amdgpu_gfx.h` (for the prototype) and `amdgpu_gfx.c` (for the implementation). And then, our process involved:
- Adding a generic function to `drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c` by removing the version-specific prefixes.
- Adding the function prototype to `drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h` to make it visible to other files.
- Updating `gfx_v11_0.c` and `gfx_v12_0.c` to call the new helper instead of maintaining 52 repeated lines;
  
Below is the diff of our implementation:

```bash

--
 drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c | 55 +++++++++++++++++++++++++
 drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h |  4 ++
 drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c  | 48 +--------------------
 drivers/gpu/drm/amd/amdgpu/gfx_v12_0.c  | 48 +--------------------
 4 files changed, 61 insertions(+), 94 deletions(-)

diff --git a/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c b/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c
index 2956e45c9..3ac7d9305 100644
--- a/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c
+++ b/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c
@@ -34,6 +34,7 @@
 #include "amdgpu_xcp.h"
 #include "amdgpu_xgmi.h"
 #include "amdgpu_mes.h"
+#include "amdgpu_userq_fence.h"
 #include "nvd.h"

 /* delay 0.1 second to enable gfx off feature */
@@ -2684,3 +2685,57 @@ void amdgpu_debugfs_compute_sched_mask_init(struct amdgpu_device *adev)
 #endif
 }

+int amdgpu_gfx_eop_irq(struct amdgpu_device *adev,
+                            struct amdgpu_irq_src *source,
+                            struct amdgpu_iv_entry *entry)
+{
+       u32 doorbell_offset = entry->src_data[0];
+               u8 me_id, pipe_id, queue_id;
+               struct amdgpu_ring *ring;
+               int i;
+
+               DRM_DEBUG("IH: CP EOP\n");
+
+               if (adev->enable_mes && doorbell_offset) {
+                       struct xarray *xa = &adev->userq_doorbell_xa;
+                       struct amdgpu_usermode_queue *queue;
+                       unsigned long flags;
+
+                       xa_lock_irqsave(xa, flags);
+                       queue = xa_load(xa, doorbell_offset);
+                       if (queue)
+                               amdgpu_userq_fence_driver_process(queue->fence_drv);
+                       xa_unlock_irqrestore(xa, flags);
+               } else {
+                       me_id = (entry->ring_id & 0x0c) >> 2;
+                       pipe_id = (entry->ring_id & 0x03) >> 0;
+                       queue_id = (entry->ring_id & 0x70) >> 4;
+
+                       switch (me_id) {
+                       case 0:
+                               if (pipe_id == 0)
+                                       amdgpu_fence_process(&adev->gfx.gfx_ring[0]);
+                               else
+                                       amdgpu_fence_process(&adev->gfx.gfx_ring[1]);
+                               break;
+                       case 1:
+                       case 2:
+                               for (i = 0; i < adev->gfx.num_compute_rings; i++) {
+                                       ring = &adev->gfx.compute_ring[i];
+                                       /* Per-queue interrupt is supported for MEC starting from VI.
+                                       * The interrupt can only be enabled/disabled per pipe instead
+                                       * of per queue.
+                                       */
+                                       if ((ring->me == me_id) &&
+                                               (ring->pipe == pipe_id) &&
+                                               (ring->queue == queue_id))
+                                               amdgpu_fence_process(ring);
+                               }
+                               break;
+                       }
+               }
+
+               return 0;
+
+}
+
diff --git a/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h b/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h
index a0cf0a3b4..a180d1903 100644
--- a/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h
+++ b/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.h
@@ -664,6 +664,10 @@ void amdgpu_gfx_csb_preamble_end(u32 *buffer, u32 count);
 void amdgpu_debugfs_gfx_sched_mask_init(struct amdgpu_device *adev);
 void amdgpu_debugfs_compute_sched_mask_init(struct amdgpu_device *adev);

+int amdgpu_gfx_eop_irq(struct amdgpu_device *adev,
+                            struct amdgpu_irq_src *source,
+                            struct amdgpu_iv_entry *entry);
+
 static inline const char *amdgpu_gfx_compute_mode_desc(int mode)
 {
        switch (mode) {
diff --git a/drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c b/drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c
index 5097de940..767887d7d 100644
--- a/drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c
+++ b/drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c
@@ -6494,53 +6494,7 @@ static int gfx_v11_0_eop_irq(struct amdgpu_device *adev,
                             struct amdgpu_irq_src *source,
                             struct amdgpu_iv_entry *entry)
 {
-       u32 doorbell_offset = entry->src_data[0];
-       u8 me_id, pipe_id, queue_id;
-       struct amdgpu_ring *ring;
-       int i;
-
-       DRM_DEBUG("IH: CP EOP\n");
-
-       if (adev->enable_mes && doorbell_offset) {
-               struct amdgpu_usermode_queue *queue;
-               struct xarray *xa = &adev->userq_doorbell_xa;
-               unsigned long flags;
-
-               xa_lock_irqsave(xa, flags);
-               queue = xa_load(xa, doorbell_offset);
-               if (queue)
-                       amdgpu_userq_fence_driver_process(queue->fence_drv);
-               xa_unlock_irqrestore(xa, flags);
-       } else {
-               me_id = (entry->ring_id & 0x0c) >> 2;
-               pipe_id = (entry->ring_id & 0x03) >> 0;
-               queue_id = (entry->ring_id & 0x70) >> 4;
-
-               switch (me_id) {
-               case 0:
-                       if (pipe_id == 0)
-                               amdgpu_fence_process(&adev->gfx.gfx_ring[0]);
-                       else
-                               amdgpu_fence_process(&adev->gfx.gfx_ring[1]);
-                       break;
-               case 1:
-               case 2:
-                       for (i = 0; i < adev->gfx.num_compute_rings; i++) {
-                               ring = &adev->gfx.compute_ring[i];
-                               /* Per-queue interrupt is supported for MEC starting from VI.
-                                * The interrupt can only be enabled/disabled per pipe instead
-                                * of per queue.
-                                */
-                               if ((ring->me == me_id) &&
-                                   (ring->pipe == pipe_id) &&
-                                   (ring->queue == queue_id))
-                                       amdgpu_fence_process(ring);
-                       }
-                       break;
-               }
-       }
-
-       return 0;
+       return amdgpu_gfx_eop_irq(adev, source, entry);
 }

 static int gfx_v11_0_set_priv_reg_fault_state(struct amdgpu_device *adev,
diff --git a/drivers/gpu/drm/amd/amdgpu/gfx_v12_0.c b/drivers/gpu/drm/amd/amdgpu/gfx_v12_0.c
index 65c33823a..aadebb4d2 100644
--- a/drivers/gpu/drm/amd/amdgpu/gfx_v12_0.c
+++ b/drivers/gpu/drm/amd/amdgpu/gfx_v12_0.c
@@ -4846,53 +4846,7 @@ static int gfx_v12_0_eop_irq(struct amdgpu_device *adev,
                             struct amdgpu_irq_src *source,
                             struct amdgpu_iv_entry *entry)
 {
- #[...] the duplicate function body from gfx_v11_0_eop_irq
+       return amdgpu_gfx_eop_irq(adev, source, entry);
 }

 static int gfx_v12_0_set_priv_reg_fault_state(struct amdgpu_device *adev,
```

## Build Process and Validation

After implementing the changes, we proceeded to compile the kernel. Our first attempt using `kw build` failed, so we performed a full configuration using `kw config` following the Tutorial 2 Building and booting a custom Linux kernel for ARM using kw:

```bash
/home/lk_dev/activate.sh
cd ~/linux-repositorio
kw init
make ARCH=arm64 defconfig
kw config build.arch 'arm64' 
kw config build.cross_compile 'aarch64-linux-gnu-'
kw build
```

During this stage, we realized the build was not actually including our changes. To verify this, we intentionally introduced a syntax error (removing a semicolon ";"), but the build still completed without errors. We then tried `kw build --menu` to locate the GFX driver, but could not find it. Consequently, we decided to edit the specific `.config` file manually used by the course's AMD CI  (available at the: <https://download.01.org/0day-ci/archive/20260322/202603221553.GzXoxzCw-lkp@intel.com/config>). We initially tried to compile for ARM64:

```bash
make W=1 ARCH=arm64 SHELL=/bin/bash drivers/gpu/drm/
```

This failed due to an architecture mismatch. We then switched the architecture to x86_64:

```bash
make W=1 ARCH=x86_64 SHELL=/bin/bash drivers/gpu/drm/
```
Once the compilation succeeded, we validated the patch style using the kernel's built-in script:

```bash
git format-patch -1 --stdout | ./scripts/checkpatch.pl –
```
The script returned the following warnings:
```bash

WARNING: Prefer a maximum 75 chars per line (possible unwrapped commit description?)
WARNING: line length of 101 exceeds 100 columns
#82: FILE: drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c:2725:
+/* Per-queue interrupt is supported for MEC starting from VI.
WARNING: line length of 101 exceeds 100 columns
#83: FILE: drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c:2726:
+* The interrupt can only be enabled/disabled per pipe instead
WARNING: Block comments should align the * on each line
```

We decided to bypass these warnings for the time being, as the line-length issues were part of the original code we relocated and not a result of new logic errors.

## CI Testing and Patch Submission

To submit our patch via email, there was an error message and we discovered we were missing the necessary SASL authentication. We resolved this by installing the following authenticator:

```bash
perl -MCPAN -e 'install Authen::SASL::Perl::OAUTHBEARER'
```
Initially, we encountered issues with the course's CI pipeline. After notifying Marcelo, he resolved a pipeline configuration error, and our patch successfully passed the testing phase. You can view the successful pipeline available at the <https://gitlab.freedesktop.org/marcelomspessoto/dsl-linux/-/pipelines/1646815/>. The submission process triggers a local CI where the patch is applied to the corresponding kernel tree, compiled, and booted into a Virtual Machine (VM). The CI consists of two main stages: 
- Build: responsible for applying the patch and compiling the kernel image 
- Test: loads the image and executes a basic boot cycle test.
  
Our patch, as processed by the course CI, is available on the public lore archive <https://lore.kernel.ime.usp.br/kernel.ime/20260415195601.160563-1-erick.am@usp.br/T/#u>

## Feedback from the Maintainers
For the final submission to the upstream maintainers, we sent our patch at 6:00 AM to align with their time zones. As a result, we received a response on the same day from Christian König, who provided the following feedback:

>"The coding style here looks completely broken. Additional to that the separation was intentional [...] What we could do is to move this chunk into a common function, there should be multiple copies of it in the SDMA code as well."

The feedback suggests that our refactoring attempt fell into two common kernel development categories:
- **Driver Forking (T1)**: Code duplication is sometimes intentional. Entire drivers or functions are cloned so they can serve as independent baselines for future hardware-specific changes.
- **Readability over Deduplication (T2)**: Maintainers often prefer duplicated code if it keeps a file self-contained, reducing the cognitive load of jumping between multiple files.
  
Regarding the comment "The coding style here looks completely broken," we were initially surprised since we had copied the functions directly from the existing files. However, moving code often exposes legacy indentation issues or requires adjustments to fit the standards of the new host file.

### Next Steps
We are encouraged by Christian’s suggestion to extract a specific logic chunk, the one involving `userq_doorbell_xa`, into a common function, as it is also used in the SDMA code. We are now preparing to move forward with a second patch based on this expert guidance.
