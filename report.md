# PES-VCS Report

## Phase 5: Branching and Checkout

### Q5.1
A `pes checkout <branch>` implementation would first update `.pes/HEAD` to point to the target branch ref, then read the commit hash stored in `.pes/refs/heads/<branch>`. After that it would reconstruct the working directory from the target commit's tree: create files that exist in the target branch, restore their contents and modes, and delete tracked files that are no longer present. The operation is complex because it must recurse through nested trees, avoid overwriting uncommitted work, and keep the index, refs, and working tree consistent.

### Q5.2
To detect a dirty working directory, compare each tracked path in the index against the current filesystem state. If the file's mtime or size differs from the index, hash the file contents and compare that hash to the staged blob hash. If the file is modified locally and the same path has different content or existence in the target branch tree, checkout must refuse. In practice, this uses the index as the baseline and the object store to compare against both the current staged blob and the target tree snapshot.

### Q5.3
In detached HEAD, new commits are created normally, but no branch ref is advanced to point to them. The commits remain reachable only through HEAD until the user moves away, at which point they can become unreachable. A user can recover them by creating a branch pointing at the detached commit before leaving that state, or by locating the commit hash and reattaching a branch to it later.

## Phase 6: Garbage Collection and Space Reclamation

### Q6.1
Garbage collection should start from all live refs in `.pes/refs/heads` and from HEAD, then traverse reachable commits, trees, and blobs recursively while marking visited hashes in a hash set. After marking, scan `.pes/objects` and delete any object not in the reachable set. With 100,000 commits and 50 branches, the number of unique objects visited is roughly the number of distinct reachable commits plus their trees and blobs, not 50 times that amount. The exact count depends on sharing, but each reachable object is visited once.

### Q6.2
Running GC concurrently with commit creation is dangerous because commit creation writes objects in stages: blobs and trees first, then the commit object, then the ref update. If GC runs in the middle, it can see the newly written objects as unreachable and delete them before the ref points to them. Git avoids this with locking and conservative retention so GC does not race with ref updates or object creation.
