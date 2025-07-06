
A few months back, I wrote an article covering the initial stages of the 'Build Your Own Git' challenge from CodeCrafters — including commands like `git init`, `cat-file`, and `hash-object`. You can find the article at this link and at the home page as well.
Since that article received good response ( even from the [CodeCrafters](https://app.codecrafters.io/catalog) as well 😃 ), I’ve been continuing the challenge at my own pace between work, and while the next full article is still in progress, I thought I’d share some raw notes and learnings for anyone following along. Since, I have finally completed the next 3 challenges, below are the notes that I made while I was making my way through these challenges. I am sharing them here in case they help others working through the same challenge.

## Pre-requisites

When a file is computed by git, it's known as a "blob"
When a directory is computed by git, it's known as a "tree"
All the content is stored as tree and blob objects, with trees corresponding to UNIX directory entries and blobs corresponding more or less to in-nodes or file contents. 

### Tree Objects
Consider the root folder of your any project. When we initialises our folder with `git init` command, it adds all the folders and files into the ".git/objects" dir either as a 'tree' or as a 'blob'. This makes the root folder of our project, a "Tree".

Now this single tree object in git, contains one or more entries each of which is a SHA-1 Hash of a blob or its sub-tree.
Each tree object contains:
	* A SHA-1 hash of the blob ( file ) or the sub-tree ( sub-folder )
	* The name of the file/directory
	* The mode of the file/directory
	  Mode represents the UNIX file system permissions. For ex:
		- `100644` (regular file)
		- `100755` (executable file)
		- `120000` (symbolic link)
		- `40000`  (directories)

For example, if you had a directory structure like this:
```
root_folder/
	file1
	dir1/
		file1_in_dir1
		file2_in_dir1
	dir2/
		file1_in_dir2
```

The entries in the tree object would look like this:
```
40000 dir1 <tree_SHA_1>
40000 dir2 <tree_SHA_1>
100644 file1 <blob_SHA_1>
```
Notes:
* The entries in a tree object are alphabetically sorted and all of this is stored in a one single line, which can be pretty-printed by the command `git ls-tree`.
* In a tree object file, the SHA-1 hashes are not in the hexadecimal format. It means, after the hash creation, we need to decode the string into raw bytes format ( 20 bytes long ).
### git `ls-tree`
`git ls-tree` command is used to inspect the tree structure. For a directory structure like above, it will output the following:

```
40000 tree <tree_SHA_1> dir1
40000 tree <tree_SHA_1> dir 2
100644 blob <blob_SHA_1> file
```
Note that the output is also alphabetically sorted.

When provided with the `--name-only` flag, it only outputs the name of the entries under that tree object like this:
```
dir1
dir2
file1
```
And, this is our next challenge: replicating the behaviour of this command:
`git ls-tree --name-only <tree-SHA>`

By the way, if we run the below command under our `codecrafters-git-go` dir
command: 
`$ git cat-file -p master^{tree}

you should see something like this:
![[Pasted image 20250601172620.png]]

The `master^{tree}` syntax specifies the tree object that is pointed to by the last commit on your `master` branch.

Reading the contents of a "tree" object is very similar to what we implemented in the `cat-file` command.
We have to repeat all the steps from `cat-file` command till we get the contents of the "tree" object after decompressing the file at the location.
If you print the contents of this tree object in the 'string' format, you should see something like this:

`tree 99\x00100644 dooby\x00\x8d\xa7Q}\xa3u1wo\x84r\x8c\xf14\xff$\xbb\xf14H40000 monkey\x007ݞ\xf0\xceJ\x8dl>\xd1uS\x80\xe4xǷ[0P40000 scooby\x00?>\xa7\xe0\x1c\xf6\b\xf9\xac\x1f\xb5.\xce־p\xec\xfa\x82~`

The header of the above string `tree 99` depicts the length of the content of our tree object in byte format.

All of this content is stored in one single line. To complete this challenge, we need to pretty-print the names of the objects under this tree in separate lines. From this string, we can see that there are 3 objects under this tree namely:
'dooby', 'monkey', and 'scooby' with modes '100644', '40000', and '40000' respectively depicting that the objects are 'file', 'dir', and 'dir'.
Note that, these are stored in alphabetically sorted order.

To fetch the names of the objects from this string, we can iterate over it, starting after the first 'null byte', which marks the beginning of the actual content for this tree.

`actualTreeContent := decompressedData[nullIndex+1:]`

As we can see, the string starts from the "mode" of the object, followed by a "space", then the "objectName" --> another "null byte" --> "SHA-1 Hash of the object", and this continues.
To keep track of all 3 data-points i.e, "mode", "name", and "SHA-1 Hash", we'll initialise 3 separate arrays.

```
var objectNames []string
var modes []string
var objectSHAs []string
```

Start by looping through the string, we'll find the index where we next encounter an "empty space" or " ".
`mode` will be everything before the "space index".
Append the `mode` string in the `modes` array.
The new content will be the remaining content.

```
spaceIndex := bytes.IndexByte(actualTreeContent, ' ')
mode := string(actualTreeContent[:spaceIndex])
modes = append(modes, mode)

actualTreeContent = actualTreeContent[spaceIndex+1:]
```
Next up, we'll compute the `objectName`.
Start by finding the index of the next null byte.
`objectName` will be the string before the "null byte".
Append the `objectName` in the `objectNames` array.
Mutate the content with the remaining string.

```
nextNullIndex := bytes.IndexByte(actualTreeContent, 0)
objectName := string(actualTreeContent[:nextNullIndex])
objectNames = append(objectNames, objectName)

actualTreeContent = actualTreeContent[nextNullIndex+1:]
```
We can expect that the rest of the string will either be the hash of the previous object that we computed. 
Get the next 20 characters from the `actualTreeContent`, which will be our SHA-1 hash string of the previous object.
Append the `currentSHA` in the `objectSHAs` array.
Mutate the content with the remaining string.
Run this loop for the rest of the string.

```
if len(actualTreeContent) < 20 {
	fmt.Fprintf(os.Stderr, "Unexpected end, Invalid SHA: %s\n")
	os.Exit(1)
}
currentSHA := actualTreeContent[:20]
objectSHAs = append(objectSHAs, string(currentSHA))

actualTreeContent = actualTreeContent[20:]
```
For this challenge, we only need to print out the names of the objects in separate lines, and our test cases will be passed.
```
for _, name := range objectNames {
	fmt.Printf("%s\n", name)
}
```

### git`write-tree`

The `git write-tree` command creates a tree object from the current state of the "staging area". The staging area is a place where changes go when you run `git add`.
The output of `git write-tree` is the 40-char SHA-1 hash of the tree object that was written to `.git/objects`.
For our challenge, we need to consider that all the files created by the tester will be in the staging area and we have to print the 40-char SHA-1 hash to stdout.

The tester will also verify that the output of your program matches the SHA-1 hash of the tree object that the official `git`implementation would write.

The implementation for this will require a recursive solution as we have to traverse through all the directories and sub-directories to be able to create the parent's directory tree object.

Let's start by deriving a simple logic for our function:
1) Start by creating an empty byte slice/array, to keep track of all the entries in our tree. This byte slice will be used to create the final hash of the current tree ( current tree means every dir that we'll encounter in our root dir ), finally returning the tree hash of our root dir.
2) We'll start with the root directory ( as we are considering all the files in our dir is in staging area ).
3) Loop through all the entries in our root directory.
4) Inside the loop, If the entry is a file/blob, we'll follow the usual steps:
	1) Creating the hash of the file.
	2) Writing the compressed version of the file in `.git` dir, at the file location derived from the hash.
	3) Create the entry of this file in our tree slice/array. This entry will be in the same format as mentioned above i.e, `\00 <mode_of_entry> <name_of_entry> \00 <raw_hash_of_the_entry>`
	4) Append this entry into our initial tree slice containing entries from all other entries.
5) If the entry is a dir, we'll follow these steps:
	1) We'll call our function again, but now with our new dir path.
	2) We should expect the above function call to return the tree hash of our dir that we encountered.
	3) Same as above, we'll append an entry for this tree object in our final tree slice, which will be later used to generate the tree hash.
6) Once we are outside the loop, it means that we have processed all the entries of our current dir/tree. 
7) Append the required header for our tree content as prefix in our tree byte slice i.e, 
   `tree \00len(<tree_byte_slice>) ...<tree_slice>`
8) Generate the SHA hash from the above content.
9) Store the compressed version of the above content as a tree object in `.git` dir.
10) Return the generated hash from the above tree object.

Call the above function in our main function, and print the hash in stdout to complete the challenge.

Here's the code for the above function:
```go

func BuildTree(filePath string) string {
	var tree []byte
	entries, _ := os.ReadDir(filePath)
	
	var entryNames []string
		for _, e := range entries {
		if e.Name() == ".git" {
		continue
		}
		entryNames = append(entryNames, e.Name())
	}
	
	sort.Strings(entryNames)
	
	for _, name := range entryNames {
		fullPath := filepath.Join(filePath, name)
		info, _ := os.Stat(fullPath)
		if info.IsDir() {
			subTreeHash := BuildTree(fullPath)
			rawHash, _ := hex.DecodeString(subTreeHash)
			entry := fmt.Appendf(nil, "40000 %s\x00", name)
			tree = append(tree, entry...)
			tree = append(tree, rawHash...)
		} else {
			content, _ := os.ReadFile(fullPath)
			blobHash := CalculateGitObjectHash(content)
			rawHash, _ := hex.DecodeString(blobHash)
			dirName := blobHash[:2]
			fileName := blobHash[2:]
			path := filepath.Join(".git", "objects", dirName, fileName)
			os.MkdirAll(filepath.Dir(path), 0755)
			writeCompressedObject(path, content, "blob")
			entry := fmt.Appendf(nil, "100644 %s\x00", name)
			tree = append(tree, entry...)
			tree = append(tree, rawHash...)
		}
	} 
	
	header := fmt.Sprintf("tree %d\x00", len(tree))
	full := append([]byte(header), tree...)
	treeHash := sha1.Sum(full)
	treeHashStr := fmt.Sprintf("%x", treeHash[:])
	dir := filepath.Join(".git", "objects", treeHashStr[:2])
	file := filepath.Join(dir, treeHashStr[2:])
	os.MkdirAll(dir, 0755)
	writeCompressedObject(file, tree, "tree")

	return treeHashStr
}
```

### git`commit-tree`


Our next challenge in the Build your own git module is creating a commit object. Creating a commit object is quite similar to creating a blob or a tree object.
The content of a normal commit object looks something like this:
```
tree 3af2624bd464b7e479432b16a89af54b82321df3
parent a7eef0abf4d3beda4a553e90c97b1ee85789d6ab
author Sarthak Duggal <duggal.sarthak12@gmail.com> 1751751580 +0530
committer Sarthak Duggal <duggal.sarthak12@gmail.com> 1751751580 +0530

Added list-tree, write-tree, commit-tree methods
```
The first line is the word "tree" followed by the SHA-1 hash of the tree object created from the files in the staging area.
The word "parent" followed by the SHA-1 hash of the parent commit.
"author" & "committer" being self-explanatory.

Atlast, the commit message after a line break.

### Wrapping up

You can find the code for this challenge at this [link](https://github.com/sardug10/make-your-own-git).

Feel free to share any feedback on my socials. You'll find the links for those at the homepage of this site.

Also, shoutout to CodeCrafters for this challenge. If you're curious to try the CodeCrafters platform yourself, here's a [link](https://app.codecrafters.io/join?via=sardug10) with a 40% discount for all subscriptions. It’s a challenge I’d genuinely recommend!
