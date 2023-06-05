
A web interface and something with uploading zips! 
After long hours of waiting, `archive-bunker` finally presents the first web-challenge in the defcon qualifiers! So let's dive straight in and see what this thing does.


# Exploration

We get website with some doomsday-paranoid advertising for storing your CI / CD artifacts in super-secur bunker. 
We interact with the application through what looks like a military-grade rugged computer terminal with shitty control buttons.
We can upload `zip` or `tar` archives by drag'n'dropping them into the screen and then we can browse the uploaded archives and see the contents of the contained files.
We can also hit a mysterious `prep-artifacts` button that will make a `app-main--<timestamp>` archive appear, containing some weird text files.

So far so ... good? Let's look at the code! HTTP requests are so web2, of course in a defcon challenge, everything happens with websockets!
Once you open the site, a websocket connection is established to provide the actual functionality. 
The back-end server is written in go-lang and handles - among others - the following websocket events/commands:

### `upload` 
As the name suggests, this command allows us to upload archives, some important observations:
- Your file name must end with `.zip` or `.tar`.
- You can not "overwrite" an archive with the same name.
- Your archive will be opened by the server and its members stored into a zip archive **with the extension stripped** 
- All uploaded archives get "censored" in the `compress_files` function. 
Meaning if an archive members name contains certain words, it is dropped and the file contents are scanned for certain regex patterns which are replaced with `***`. The words and patterns are specified in the config file.

### `download` 
So say you uploaded `my-archive.(tar|zip)`, you can then download members from your archive by passing `my-archive/my-member` to the `download` command. (Note the lack of extension!) 
To accomplish this "feature" the server will open the **zip** archive created in the upload step. 


### `job` (packaging) 
The third important command is the `job` command, which only supports one subcommand: `package`. 
You can also pass an arbitrary `name` for the package-job.
Under the hood, this feature is massively over-engineered:
In the spirit of CI-Tools the actual input is a yaml file that describes `steps` (archives) of a certain `name` that contain certain files.
The go server will prepare that yaml by applying some variables to a template file (included in the source) with go-langs [templating engine](todo: link) which is similar to jinja or django templating for the web folks reading this.
It will create a `.tar` archive and again a zip archive **without** a filename extension for every `step`.

Did you catch it? It creates archives with a `name` and we can specify a `name` for our websocket command...

# It's an InYection
So we control the `name` variable and the template for the CI job looks like this:
```
# Todo 
```
with our specially crafted `name`, it may turn into something like this:
```
# Todo, with injected name
```
One limitation we have here, is that we can only include files into our archive that are inside the `/project` directory. This is again specified in the config file via the `root` setting. 
The good news is that `flag.txt` is in this `/project` directory and that the CI job will create a `.tar` archive (which has no compression) and a zip archive in the `/data` directory.
The bad news is that the zip archive will be created with `compress_files`, which applies the "censorship" mechanism described earlier. 
And you might remember that the download command will always and only look at zip archives...

So we have a mechanism to get the flag into a tar file in `/data`, but how do we get it out of there?

<10 Hours later>

# There is an overwrite

We spent a lot of time pocking around for path traversals in the archive processing logic this is a web-challenge, right? RIGHT?

We didn't find anything like that, but suddenly we discovered that the loathed `compress_files` function did have a bug: 

It didn't check whether the file it was writing an archive to already existed! So it would overwrite existing files. 
Notice that it will overwrite not replace: If the original file is larger than the zip file, the "bottom" contents of 
the original will not be erased! Additionally, we can use this to write into the tar created by the InYection by using 
`filename.tar.zip` as the filename, since only the `.zip` extension will be stripped!

So now it became clearer what we had to do: 
1. Create a large tar with the flag inside using our InYection
2. Use the overwrite to write zip metadata into the tar archive
3. Read the flag from zip/tar with the download function

Our thought process was of course a lot less straight forward than that during the CTF. But to spare you the pain, we 
will now give you a short intro into the zip archive format:

<TODO>
Important to mention:
- Directory header (AT THE BOTTOM of the file!)
- Member header
- Compression method field
- Cksum (Especially:Cksum == 0)
- Member name field (Not compressed, arbitrary length, arbitrary data)
- Extra field length
</TODO>

# Zip! Y u so nasty??? 
The keen reader might have observed that the format allows for some nasty ambiguities: Some header fields, like 
cksum and compression method are in the directory header AND member header, so which one takes precedence? 
We just checked the go implementation and it seems like it will only consider the values in the directory header for 
all fields, except for `extra-field-length`, which makes a lot of sense, since ignoring extra fields in the member 
header might otherwise cause an invalid read. This field is also a key part of our exploit: By setting the value very 
high, we can make the go zip reader read "below" the contents of our actual zip file -> meaning the old contents of the 
tar archive -> meaning the flag!

Easy! Right? No. The trouble is that we can not control the zip metadata that `compress_files` will write. It will just 
open the provided archive and copy the members into a new zip archive with defaults for compression, 
extra-field-length, etc. In addition: The `download` function will panic and give us an empty file if the go zip reader 
encounters anything it doesn't like, for example invalid checksums or broken directory headers. 

But truth be told, we do control one of the header fields: The member name! After a lot of head scratching our refined 
exploit plan looks like this:

1. Create a large `notflogs.tar` with the flag inside using our InYection
2. Write a smaller `small.zip` into the tar, so we have a zip directory header
3. Write an even smaller `tiny.zip` into the tar, with a member name that:
   - Contains a valid member header with a large `extra-field-lengt` to make go zip reader jump down and read the 
   contents of the tar  
   - Overrides the `compression-method` and `crc` in the directory header to `0` 
   - Overrides the `member offset` to point to the member header described above
   - Does not break the headers in a way that upsets the go zip reader 
3. Download the flag from our crafted zip.

< Meme about defcon "web" challenges >







