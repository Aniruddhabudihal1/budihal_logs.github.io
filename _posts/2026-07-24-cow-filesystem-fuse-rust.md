This write up is an article on the internals of a copy on write filesystem and how you can build one in rust, you can check out my implementation here : [unionFS](https://github.com/aniruddhabudihal1/unionFS).
It is an implementation of the mulithreaded copy on write filesystem in rust using the fuser library. While researching for the project, I found that there was very minimal or no resources available online on building something like this in Rust, so I decided to write one :)
There has been no use of AI in either writing the code, or anything in this article

## To start of, what is a copy on write file system and why is it needed ?

I like to explain stuff using analogies so you could think of a copy on write file system kind of like a polarizer (ya I know not the first thing that you think of when talking about filesystems). A polarizer allows you to view particular light rays at particular angles, but how does it work ?
It basically kind of like a slit over the light source and filters out all the other stray light rays that are coming in from different angles and allows one particular angle of light rays to pass through.

![A simple illustration of how polarizer works](/assets/img/polarizer.png)

A simple illustration of how polarizer works, where it is allowing light rays of only particular angles to pass through (again you could think of this as a sieve for light rays)
(side track but if you want to see a real world example of this, this is used in cars where you can see infotainment screens as a passenger but not as a driver driving : [this is a real world example of this](https://youtu.be/1gmjRFKAoA4?si=E3zJjZYc5xwDQC10&t=75))

But what does this have to do with a copy on write filesystem ?
In a copy on write filesystem, each instance interacting with the file system gets its own unique view at the file system, you can think of it as each instance holding their own angle that they use to filter the light source and they end up seeing and working with only those particular angles.

So an instance interacting with a COW filesystem can edit / create / delete a directory or a file or a symlink, and it will be reflected only in its instance of the filesystem.
So, are we just duplicating the whole file system every time a new instance interacts with it ? Well that would work if you have infinite storage, but on any generic system thats not feasible, so we need a system that kind of gives a translucent view of the filesystem to the user where you see stuff you are supposed to see and not another instances stuff.

This is achieved by using a base readable on which you are mounting your file system on. If a particular instance has just read a particular directory or a file without making any modification to the original file, it exists solely in the readable from which every instance can read from (if they also have not made any modification to the same content). Upon any modification to the original content the content will be reproduced on the writable path for that particular instance on which it will have exclusive access.

![an image to visualize a copy on write filesystem](/assets/img/final.png)

In the above diagram, every node which is green in colour is a node in the base readable path, the writable instance clones the directory structure in the readable path which is indicated by the purple colour node, and every write or insertion done will result in the formation of a new node in just the writable path which is indicated by a red node.

The newly cloned filesystem on the right, shows how a bare bones cloned fs will look like when there are no new nodes, or any existing nodes edited

And also an inode is the unique value associated with a particular node, the node could be a directory or a file. So globally throughout the file system every node regardless of whether it is under the writable path or the readable path will have a globally unique indoe value

### where is a COW filesystem used ?

Docker uses a copy on write filesystem to manage its image files. For example lets say you had 10 containers running an nginx instance on a single system. Having a nginx instance for each container will lead to a lot of space being used and wasted, hence here a copy on write method is used where there is a base image layer, and every edit is translated only in the respective containers writable path.
Even at the fundamental OS level when you execute a fork call, because the child process created contains almost the same state as the parent. The scale at which fork calls are called, we would very easily run out of memory if we were to make separate memory pages per new process. so here again this is dealt where every new process that is created uses a copy on write method with its parent that created it, every edit done to a memory page will only then result in cloning of the memory page for the child.

##  Implementing a COW filesystem

Before we understand how to implement a COW filesystem, we need to understand the internals of a normal filesystem and how it all works. In a normal file system whenever you do an operation like `cd` or `ls` or creating or deleting a file, etc all of the calls are routed to the OS kernel which then runs the required functions to get the result.
Some of these functions are : 
- lookup : it searches for the inode value of a particular node name and returns the inode value if it exists
- readdir : it reads the nodes present under a particular directory
- getattr : it gets the attributes of a particular node
- write : This write the required content into an already existing file, or a newly file that is to be created
- open : opens a node based on the flags provided, and returns the new file handle for that particular file
- read : reads the contents of a file and returns the content of that file
- rename : renames the node, or can move the directory to any other location in the filesystem
- rmdir : removes a directory
- mkdir : creates a new directory 
- create : creates a new file

Most of the above functions significantly change when dealing with a copy on write file system, as you need to account for a lot more things to get the logic working
For implementing your own functions for a file system we need to first redirect the call to operations like `ls` or creating or deleting a  node away from the kernel logic to our own logic in the user space. This is achieved by using a FUSE library which redirects all interactions with the mounted file system to your userspace function logic. For my implementation I used the `fuser` library which provides the fuse library in Rust. 

Before starting of implementing the functions, we need to first understand how we are building the base readable path. The base readable path will be the path on which we are mounting our filesystem. So before even calling the mount, we need to read through the file tree and map it out. I have just kept the state of my readable path in memory, because I am not going to be using it on very large files, but if you were you should probably write it to disk.
So I have a function called `instantiate_fs` which recursively goes through each directory in each path and maintains a state of the mounted path (by state I mean the file content and the children of each directory and the general mapping of the all the nodes).

```rust
fn instantiate_fs(file_system_instance: &UnionFs, path: &PathBuf, parent_inode_value: u64)
```

I then go through each instance of the directory passed in and traverse through each of its children giving each one of them a new inode value and adding it to my : `name -> inode mapping` (under the parent directory) and in the global state the `inode -> node mapping`, and also to my global `inode -> name` mapping

```rust
{
    let next_val = file_system_instance.curr_inode_val.try_write().unwrap();
    increment_global_inode_val(next_val);
}

let mut global_state = file_system_instance
    .inode_to_content_mapping
    .try_write()
    .unwrap();
let mut inode_to_string_mapping = file_system_instance
    .inode_to_string_mapping
    .try_write()
    .unwrap();
inode_to_string_mapping.insert(
    *file_system_instance.curr_inode_val.try_read().unwrap(),
    str_child_path.clone(),
);
if let Some(parent_inodeinstance) = global_state.get_mut(&parent_val) {
    let mut parent_hash = parent_inodeinstance
        .node_kind
        .children_of_directory()
        .unwrap()
        .try_write()
        .unwrap();
	
    parent_hash.insert(
        str_child_path.clone(),
        *file_system_instance.curr_inode_val.try_read().unwrap(),
    );
}
```

Then dependening on whether the current node is a file, or a directory we create a new node instance to which you add the relevant node attributes and its content.If it is a directory then this function is called recursively once again on that directory
At this stage we mount the filesystem, and every subsequent operation done on the mounted path is redirected to our filesystems function which we must now write 

But before getting into functions, we need to identify how we will identify each unique instance that interacts with our filesystem ? We cannot use something like a process id, as these change frequently and we will end up creating a new writable path every time it changes.
So it depends on what you choose to define as an "instance" that is interacting with your system. In the case of Docker a container is an instance, but for me to test it out I am going to just be creating a new terminal instance and interact with the file system. So I used a  `tty_nr` value which is unique to each terminal instance that interacts with the file system, even if the pid changes the `tty_nr` remains constant.
So whenever we encounter a new `tty_nr` value calling our lookup function we have to clone all the directories in the readable path for that particular instance. But why just for directories, why not just leave it with a root stub which can be expanded upon later ? 
simply because it is much easier to have the base readable directory structure in place, plus directories are pretty light weight as all they do is point to other directories (atleast in the current state when I clone it, as I am not including files in the cloning process)

I have a process called `clone_fs` which when called for a session id will instantiate the readable directory path for its respective writable root

```rust
fn clone_fs(&self, writable_root_inode: u64, ses_id: i32)
```

(note : here I have refered to `tty_nr` as `ses_id` instead, which basically means that its an unique id for the particular terminal instance, I use the 2 terms interchangeably throughout the code )

```rust
if let Ok(proc) = Process::new(comm_pid) {
    if let Ok(stat) = proc.stat() {
        ses_id = stat.tty_nr;
        println!("The session id is : {}", ses_id);

        let mut ses_maps = self.session_id_mapping.try_write().unwrap();

        if let Some(res) = ses_maps.get(&ses_id) {
            println!(
                "session_id exists with the root inode val for its respective root being : {}",
                res
            );
        } else if ses_id == 0 {
            println!(
                "session id of 0 gotten, skipping cuz it does nothing and is some background process from the gvfs"
            );
            return;
        } else {
            println!(
                "session id does not exist and we have to now add it to the hash map along with the root inode for that particular session id"
            );
```

I have a global `session_id_mapping` which is the maps each unique instance to its writable root inode value. And also the session id was getting returned as 0 for some reason if any insertion or deletion was done in the root directory, which was basically the Gnu virtual file system making sure the thing existed on which I wanted to work on, but that check is redundant in my case, hence I just leave it as is and do nothing when session id is 0, and then the subsequent request that comes about contains the actual session id value (if you have noticed whenever you mount a usb stick, you get few directories called Trash-1000, .xdg-volume-info which are also searched by the GVFS with session id of 0)

### lookup function

This is probably the single most important function for a file system, cause without this you are essentially just blind to where the nodes are actually present. So the way the lookup function works is, the fuse layer receives the path which has to be worked on, which then sends one by one the name of each node in each layer to the lookup, to essentially just search for it and confirm its existence and return the result.

so if lets say I had : `/home/test/something.txt` , the fuse layer would sends the parent as `/` and searching for a child called `home`, if it exists it returns the inode attribute of the particular node, which in the subsequent lookup function `home` becomes the parent and `test` is the next child and it goes so on.

In a COW filesystem, the lookup function is the first time the file system will encounter a new session id, so we must make that check first where we check if the session id already exists or we have to clone the readable path for the new session id, in fact the code snippet above is from the lookup function. The next course of action would be to search the hashmap of the parent directory and find the child's name, but we also need to keep in mind that there is both a readable path and a writable path, and it could be that the file at the end of it could exist solely in the writable path or solely in the readable path or in both ! 

```rust
let readable_mapping = self.writable_to_readable_inode.try_read().unwrap();
if let Some(relative_readable_inode) = readable_mapping.get(&parent.0) {
    if let Some(readable_inode_val) = relative_readable_inode {
        if let Some(readable_result) =
            self.check_readable_path(*readable_inode_val, str_name.clone())
        {
            readable_inode = Some(readable_result);
        } else {
            println!(
                "[lookup] : {} does not exist in teh readable path",
                str_name
            );
        }
    } else {
        println!(
            "[lookup] : {} does not exist in teh readable path",
            str_name
        );
    }
} else {
    println!("[lookup] : parent in the readable path does not exist");
}
```

in the above code snippet, I am finding the equivalent readable instance of the inode

Because we have instantiated every single directory that existed during `clone_fs` for the writable path, we can be sure that every instance of a directory whether newly created or deleted or never used will always be present in the writable path, and we have to account only for files. So when returning the inode attributes for a directory we always return for the writable path and not the readable path, cause once we have the writable path directory inode, we can find the equivalent readable path inode (from the writable inode to readable mapping in the global state), so once we get to the file we have a couple of choices : 
- If the writable inode for the file exists : just return that without thinking about the readable inode attributes
- if the writable inode does not exist and the readable inode does exist, return the readable inode attributes
- if neither exist, then return an error

```rust
if readable_inode.is_some()
    && writable_inode.is_some()
    && writable_inode.unwrap() != 0
    || readable_inode.is_none()
        && writable_inode.is_some()
        && writable_inode.unwrap() != 0
{
    let global_instance = self.inode_to_content_mapping.try_read().unwrap();
    let attr = global_instance.get(&writable_inode.unwrap()).unwrap();
    reply.entry(&dur, &attr.inode_attributes, Generation(1));
} else if readable_inode.is_some() && writable_inode.is_none() {
    println!(
        "found {} only in readable path, so updated global state accordingly",
        str_name.clone()
    );
    let global_instance = self.inode_to_content_mapping.try_read().unwrap();
    let attr = global_instance.get(&readable_inode.unwrap()).unwrap();
    if attr.get_inode_kind() == FileType::RegularFile {
        attr.node_kind
            .update_writable_parent_instance_for_files(ses_id, parent.0);
    }
    reply.entry(&dur, &attr.inode_attributes, Generation(1));
} else {
    println!(
        "Node with the name {} does not exist",
        name.to_string_lossy()
    );
    reply.error(Errno::ENOENT);
}
```

This piece of code is the checking if a particular node exists in the readable path or writable path (or is it deleted, I denote deleted nodes by marking their inode value to 0,  but I will get to this later) and the corresponding logic

### readdir function

This function gets invoked whenever you run `ls` on a directory, so in a basic filesystem you would just list out all the contents of a directory, but with a COW filesystem you have to take into account the readable and the writable instance of a directory, and list every file that occurs in each readable or writable path just *once*. So here the parent inode returned by the fuse layer is the writable inode, because my lookup logic will return a writable instance if it exists and earlier I have established that every instance of a directory will be present in the writable path.

So here once again we use the writable to readable mapping to get the readable instance of the writable directory, there can be a case where a new directory was created and exists solely in the writable then it will not search for the readable instance. I have a vector which first gets filled by the writable directory with all of its children then, we traverse over the readable children, and if we encounter a name that already exists in the vector filled by the writable directory then it is omitted.

### write function

This function gets invoked whenever there is an insertion / editing to a file. There are plenty of scenarios to deal with : 
- File exists  in the readable path
- file exists in the writable path
- file exists in neither

To now deal with the first challenge that is when file exists only in the readable part, is a fairly difficult challenge as when you are in the write function, you dont get the opportunity to call lookup upto the writable parent instance. So we need access to the respective writable parent instance, but while working with the global state that I have access to right now. So what I did is for every node in the readable path, there is a hash map which contains a mapping of the session id to the respective writable instance of the parent for that session id. It had to be a hash map and not a simple u64 containing the writable parents inode value because, there can be multiple session id's each interacting with the single readable instance of the node. So while the `clone_fs` is being run we set the writable instance of the parent to the respective readable paths hash map of the `writable_instance_of_parent` hash map.

```rust
for (_, ii) in readable_children_iter {
    println!("invoking ii for : {}", *ii);
    let readable_instance = readable_content.get(ii).unwrap();
    let readable_type = readable_instance.get_inode_kind();
    if readable_type == FileType::Directory {
        println!(
            "We are pushing {} which has parent {:?} and session id : {} and its respective writable_root_inode is : {}",
            ii, parent_name, ses_id, writable_root_inode
        );
        readable_children_vector.push(*ii);
        readable_instance
            .node_kind
            .update_writable_parent_instance_for_dirs(ses_id, writable_root_inode);
    } else if readable_type == FileType::RegularFile {
        readable_content
            .get(ii)
            .unwrap()
            .node_kind
            .update_writable_parent_instance_for_files(ses_id, writable_root_inode);
    }
}
```

This is the code snippet from `clone_fs` which is adding the writable parent to the hash map

```rust
if let Ok(proc) = Process::new(comm_pid)
    && let Ok(stat) = proc.stat()
{
    ses_id = stat.tty_nr;
    println!("The session id is : {}", ses_id);
    let writable_parent_instance = inode_instance_of_readable_node
        .get_writable_parent_instance_for_files(ses_id)
        .unwrap();
    let tmp = global_mapper.get(&writable_parent_instance).unwrap();
    let inode_string_mapping = self.inode_to_string_mapping.try_read().unwrap();
    let name = inode_string_mapping.get(&ino.0).unwrap().clone();
```

This is the code snippet in the write function, which is calling on the hash map to get the writable parent

This does'nt feel like a very clean solution, but boy this was all I could come up with, maybe later can come with a better plan
The rest of the cases where the file already exists in the writable path, or you have to create a new one (where it does not exist in either path) are pretty straightforward where you just create a new file and then and insert the data to the file

### unlink function

This is probably the only function, which requires you to think about the logic of COW filesystem more deeply that any other logic. Cause if we were to use standard logic to delete a file while also following the roles that govern a COW filesystem, it would not actually delete the file because : in a normally we could just remove the instance of the filesystem from the parents hash map, but we cannot touch the readable path cuz its immutable. So everytime lookup function would see that the node resides in the readable path and not the writable path and assume that it was never edited, which is wrong.

So in a traditional COW filesystem like the one used by docker, a whiteout file is used to mark a particular node as deleted, where it appends `.wh` to the beginning of the file to mark that its deleted. I kinda didnt like this, like idk it didnt feel like a very clean way of dealing with it, cause having edit the name of the file just didnt feel like the correct way.
So after a while of pondering I came up with this idea, where instead of changing things like the name or adding it to a list of extended attributes, I decided to just mark the inode value of the name in the parents hash map to zero. After this I just delete the global state associated with that particular inode. This also allows me to cleanly deal with deleted nodes in my lookup logic, where I can just check if the child is pointing to 0 and then return saying the file is deleted.

```rust
if wriatble_status.is_some()
    && readable_hm.get(&name.to_string_lossy().to_string()).is_some()
    || readable_hm.get(&name.to_string_lossy().to_string()).is_some()
        && wriatble_status.is_none()
    || wriatble_status.is_some()
        && readable_hm.get(&name.to_string_lossy().to_string()).is_none()
{
    println!("enters that large if clause");
    println!();
    if let Some(val) = wriatble_status {
        old_inode = Some(*val);
    }
    writable_hm.insert(name.to_string_lossy().to_string(), 0);
    final_status = Some(true);
}
```

if its present in either or both paths, the name is linked with 0 in the parents hash map

A similar logic can be used to delete directories as well, but for other implementations of a COW filesystem used by lets say docker, they use different methods of marking a file and a directory as deleted

### rename function 

For rename the cases you have to deal with are : 
simply renaming the file or directory to something new which just involves you changing the parents hashmap and the global inode to string mapping. If the file does not exist in the writable path, you have to make its instance in its writable path

If you are moving the file or directory from one place to another, there are a lot of things to take care of like, you must check if its solely present in the writable path or readable path or both and then do accordingly. When we say "moving" a node from under one parent to another, you have to remove the instance from that old parent ; then delete the node under the old parent by marking its inode value as 0.
You might ask as to why its being marked as 0, cause if the lookup function were to search for this node in the parents hash map it would think its deleted and will always return not found. But as we are moving it to an entirely new parent, the particular name will never get searched in the old parent directory, and hence no issues.

example : if I were to move `/home/test.txt` to `/home/something/testt.txt` 
In the home directory we mark test.txt to 0, and then under the new parent we create a new inode value pointing to the new name and we redirect rest of the state of the filesystem from the old inode value to the new inode value. Like we will need to update the writable to readable path (if it was already existing in the writable path), the inode to inodecontent mapping needs to change, the inode value within the  attributes also needs to change, the old inode to string mapping needs to be removed and a new mapping added.

but all of the above can happen only if it already existed in the writable path, what if it did not ?
we would have to first mark the old name to an inode value as 0, in the old parent directory and then create a new inode value and the rest of the state required for the new node.

```rust
let child_instance = global_mapping.get(val).unwrap();
if child_instance.get_inode_kind() == FileType::RegularFile {
    readable_content = Some(
        child_instance
            .node_kind
            .get_file_content()
            .unwrap()
            .try_read()
            .unwrap()
            .to_vec(),
    );
    initial_hm.insert(name.to_string_lossy().to_string(), 0);
} else {
    old_hm = Some(
        child_instance
            .node_kind
            .children_of_directory()
            .unwrap()
            .try_read()
            .unwrap()
            .clone(),
    );
    initial_hm.insert(name.to_string_lossy().to_string(), 0);
}
```

In the following code snippet, we are marking the old name to 0

```rust
if old_hm.is_some() {
    let new_inode_content = InodeContent {
        inode_attributes: make_attribute(*self.curr_inode_val.try_read().unwrap(), true),
        node_kind: Node::Directory {
            hash_of_children: RwLock::new(old_hm.unwrap()),
            writable_parent_instance: RwLock::new(HashMap::new()),
            is_this_in_readable_path: false,
            is_this_in_writable_path: true,
        },
    };
    global_state.insert(*self.curr_inode_val.try_read().unwrap(), new_inode_content);
} else if readable_content.is_some() {
    let new_inode_content = InodeContent {
        inode_attributes: make_attribute(*self.curr_inode_val.try_read().unwrap(), false),
        node_kind: Node::File {
            file_content: RwLock::new(readable_content.unwrap()),
            is_this_in_readable_path: false,
            is_this_in_writable_path: true,
            writable_parent_instance: RwLock::new(HashMap::new()),
        },
    };
    global_state.insert(*self.curr_inode_val.try_read().unwrap(), new_inode_content);
}
```

updating of global state after deleting 

### final thoughts

This was my first significant project using rust (or otherwise as well) and really enjoyed building it, and got to learn sooo much. Like before working on this project I knew somethings here and there about filesystems, but nothing solid and concrete. Really enjoyed building this project and would recommend it to anyone who wants to understand how dockers filesystem works, or how file systems work in general.

There are a lot of things I would loved to have done a bit differently, like for example when I instantiate the readable path or when I am cloning the writable path, I am doing so recursively, and it just does not feel like a clean solution, feels very hacky for some reason, so that is something I will definitely want to revisit. 

 I have also used reader writer locks to implement concurrency, but I want to see the performance difference of using Async rust on this project. And lately a lot of my youtube has been filled with talks on how to implement data oriented design, which I would love to do just to explore, cause in the words of Richard Feynman "What I cannot create, I do not understand" and thus would love to get my hands dirty and learn about it and see with my eyes how my project interacts differently.

If you have read till here I would like to thank you for taking your time to go through this technical write up of my project.
