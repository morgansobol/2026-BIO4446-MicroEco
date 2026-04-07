# Intro to Terminal and the Command Line

This week, you'll get comfortable using the Unix command line, which is an essential skill for working with sequence data. 

---

## Let's establish some basics first. 

Unix (the "OG" operating system, found in Macs) and Linux (Unix-like) users will find a `Terminal` app already installed on their computers. If you are using Windows, I recommend downloading MobaXTerm (https://mobaxterm.mobatek.net/), but other software is available. We can also get you set up on your personal computer later if you own a Window/PC. Most bioinformatic software is written for Unix/Linux, which is why Window users need extra tools to use these programs. 

Now, the `_terminal_` is a text input and output environment where we can type commands and see the output. In other words, it is the "window" in which you enter the actual commands and those commands are interpreted and run by a `_shell_`. 

The `shell` is the program inside the `terminal` that actually processes commands and returns the output. In most Linux and Unix (Mac) operating systems, it uses a `_bash_` shell, which is essentially its own programming language and what we will use below. 

Different shells provide unique features and syntax: on macOS, the default is now Zsh (formerly Bash) with Unix-style commands, while Windows primarily uses PowerShell and Command Prompt, with options to install Unix-like shells such as WSL or Git Bash.

Think of it this way: the `terminal` is the drive-thru speaker, the `shell` is the person taking your order and relaying it to the kitchen, and the people in the kitchen cooking your meal are the `operating system`. 

<img width="2754" height="1803" alt="image" src="https://github.com/user-attachments/assets/c2607657-3643-447d-be3f-69511845ae86" />

## File structure 
Computers store file locations in a hierarchical structure. We are typically already used to navigating through this stucture by clicking on various folders (also known as **directories**) in a Windows Explorer window or a Mac Finder window. Just like we need to select the appropriate files in the appropriate locations there (in a Graphical User-Interface, or GUI), we need to do the same when working at a command-line interface. What this means in practice is that each file and directory has its own “address”, and that address is called its “path”.

Additionally, there are two special locations in all Unix-based systems, so two more terms we should become familiar with: the “root” location and the current user’s “home” location. “Root” is where the address system of the computer starts; “home” is where the current user’s location starts (this is where you should be now)

<img width="1511" height="1799" alt="image" src="https://github.com/user-attachments/assets/d051f1d5-91ee-44b7-b175-9471422c174b" />

## 🧪 Exercise 1: Navigating the Filesystem

Ok, let's explore the terminal now. 

Let's check where you are first by <ins>p</ins>rinting your <ins>w</ins>orking <ins>d</ins>irectory (i.e. where you currently are in the system).
```bash
pwd
```
It should look something like this `/home/user/` and tell you that you are in the "home" location. 

Now <ins>l</ins>i<ins>s</ins>t the contents of your current directory using the `ls` command.
```bash
ls
```
You should see hopefully see other directories like Desktop, Downloads, Documents, etc.

Often, there are also *hidden* files, typically configuration files, which begin with a dot (`.`). E.g., your bash profile is configured by the file ~/.bash_profile. Configuration files do things like store settings and preferences for programs, determine what programs are "turned on" when you log in, and customize how your shell behaves. 

To see these *hidden* files use the `ls` command with the argument (also referred to as a flag) `-a`, to see <ins>a</ins>ll files, including hidden ones.
```bash
ls -a
```
You should see file names like these `.  ..  .bash_history  .bash_logout  .bash_profile  .bashrc  .conda, etc`. 

We can also use `ls` to see the sizes of the files, in bytes, in our directories with the argument -l.
```bash
ls -l
```

I prefer to add -lh, the "h" prints the sizes in a <ins>h</ins>uman-readable format.
```bash
ls -lh
```

Ok, lets move into into the Desktop directory using the `cd` command. 

```bash
cd Desktop/
```

Now, use `pwd` and `ls` to see where you are and what is there.
```bash
pwd
```
```bash
ls
```
What do you see? 

To move back one directory to _home_ (i.e. root directory), simply use `..` like so:
```bash
cd ..
```
Now, use `pwd` and `ls` to see where you are and what is there.
```bash
pwd
```
```bash
ls
```
What do you see? 

Typing `cd` alone anywhere you are will also bring you all the way back to home. 

Ok, now let's go back to Desktop.
```bash
cd Desktop/
```

Ok, that was a brief intro to moving around the command line. Practice makes perfect, so practice this when you can, and it will eventually become natural. 

Summary of commands used:
| Command                             | Description                                                                       |
| ----------------------------------- | --------------------------------------------------------------------------------- |
| pwd                                 | Lists the path to the working directory                                           |
| ls                                  | List directory contents                                                           |
| ls -a                               | List contents including hidden files (Files that begin with a dot)                |
| ls -l                               | List contents with more info including permissions (long listing)                 |
| ls -lh                              | List content sizes in readable format                                             |
| cd                                  | Change directory to home                                                          |
| cd [dirname]                        | Change to specific directory                                                      |
| cd                                  | Change to home directory                                                          |
| cd ..                               | Go back one directory                                                             |


## 🧪 Exercise 2: Creating and editing directories + files  
> [!WARNING]
> Using commands that do things like create, copy, and move files at the command line will <ins>**overwrite**</ins> files if they have the same name. And using commands that delete things will do so <ins>**permanently**</ins>. Use caution using these commands.

So we should still be in the Desktop directory. Let's make a new directory in this one. 
Start with <ins>m</ins>a<ins>k</ins>ing a new <ins>dir</ins>ectory, using the `mkdir` command, then use `cd` to move into it.
```bash
mkdir 2026-MicroEco16S
```
```bash
cd 2026-MicroEco16S
```
```bash
ls
```
Are you in 2026-MicroEco16S?

Now to create a new file in this directory, we will use the command `touch`. Then use `ls` to check that the file is there.
```bash
touch sample1.txt
```
>[!NOTE]
> The second part of a file name is called the filename extension, and indicates what type of data the file holds. Here are some common examples:
>* .txt is a plain text file.
>* .csv is a text file with tabular data where each column is separated by a comma.
>* .tsv is like a CSV but values are separated by a tab.
>* .log is a text file containing messages produced by a software while it runs.
>* .pdf indicates a PDF document.
>* .png is a PNG image.
>* .sh indicates a shell script file.
>  
>This is just a convention, albeit an important one. Files contain bytes: it’s up to us and our programs to interpret those bytes according to the rules for plain text files, PDF documents, configuration files, images, and so on.
>Naming a PNG image of a whale as whale.mp3 doesn’t somehow magically turn it into a recording of whalesong, though it might cause the operating system to try to open it with a music player when someone double-clicks it.


It is often very useful to be able to generate new plain-text files quickly at the command line, or make some changes to an existing one. One way to do this is using a text editor that operates at the command line. Here we’re going to look at one program that does this called `nano`. Let's test it with a file that already exists.
```bash
nano sample1.txt
```

This will open up an interface and allow you to add text. Add two sample names, A and B.
```bash
sample_A
sample_B
```
To save the file and exit, we need to use some of the keyboard shortcuts listed on the bottom. Type "ctrl" + "x". It will ask if you want to save, type "y" and then press "enter". 

To get a "sneak-peak" at what we added in the sample1.txt file, we can either use the `head` command to show the top of the file contents. There is also `tail`, which prints the last 10 lines of a file by default:
```bash
head sample1.txt
```
There are a few other options to view files.
For example, the command `less` lets you scroll through a file but not edit it. Try it out on `sample1.txt`. (To exit the `less` command, just press `q`. 
```bash
less sample1.txt
```

The command `cat` will display the whole file at once, so it is better to use for shorter files.
Try it out too `sample1.txt` file if you'd like.

Lets <ins>c</ins>o<ins>p</ins>y the file and rename it. The first file called is the file you want to copy, the second is what you want to name the copy.
```bash
cp sample1.txt copy_sample1.txt
```
```bash
ls
```

Let's move this file to a new directory. Start by making a new directory called `copies`. 
```bash
mkdir copies
```

Ok, now we want to move the copy_sample1.txt file into the `copies/` directory. To do that we will use the <ins>m</ins>o<ins>v</ins>e command `mv`), but remember the file is in the previous directory (hint: ".."). 
```bash
mv copy_sample1.txt copies/
```

Let's see if we did that right using `ls`.
```bash
ls copies/
```

Lets get rid of the renamed file. To <ins>r</ins>e<ins>m</ins>ove files, use the `rm` command. First, go into the `copies` directory.
```bash
cd copies/
```
> [!TIP]
> When starting out, especially, it might be a better practice to use the -i flag with `rm`. This will prompt the terminal to first ask permission before you delete something.
```bash
rm -i copy_sample1.txt
```
```bash
ls
```

You can also remove entire directories. Let's remove the `copies/` directory using `-r` flag.
```bash
cd ..
```
```bash
rm -r copies/
```
```bash
ls
```
Commands used and other flags
| Command                     | Description                                         |
| --------------------------- | --------------------------------------------------- |
| mkdir [dirname]             | Make directory                                      |
| touch [filename]            | Create file                                         |
| rm [filename]               | Remove file                                         |
| rm -i [filename]            | Remove directory, but ask before                    |
| rm -r [dirname]             | Remove directory                                    |
| rm -rf [dirname]            | Remove directory with contents                      |
| rm ./\*                     | Remove everything in the current folder             |
| cp [filename] [dirname]     | Copy file                                           |
| mv [filename] [dirname]     | Move file                                           |
| mv [dirname] [dirname]      | Move directory                                      |
| mv [filename] [filename]    | Rename file or folder                               |
| mv [filename] [filename] -v | Rename Verbose - print source/destination directory |

## 🧪 Exercise 3: Wildcards

Now, wildcards are special characters that enable us to specify multiple items very easily. Let’s say we only want to look for files, in our current working directory, that end with the extension “.txt”. The * wildcard can help us with that.
```bash
ls *.txt
```
What this is saying is that no matter what comes before, if it ends with “.txt” we want it.

>[!TIP]
>I just want to note a few keyboard commands that are very helpful:

- `Up Arrow`: Will show your last command
- `Down Arrow`: Will show your next command
- `Tab`: Will auto-complete your command
- `history`: Will show a history of commands used in that session


## 🧪 Exercise 4: Terminus game

Practice navigating the command line by making your way through the world of Terminus. The locations you can go to are directories, which you `cd` to move back and forth, and `ls` to look around and see what is in the area. Items in each area are files, so you use `less` to open them (interact with them), or `cp` to make copies of an "item," `touch` to create an item you need, `rm` to remove an obstacle, etc. If you need help with a command, type `man` and the command you want help with. 

Follow this link to access Terminus: https://web.mit.edu/mprat/Public/web/Terminus/Web/main.html 
When it loads, follow the instructions. Have fun!

Put a sticky note up when you've successfully done all three below:
1. When you had to use the command `mv`. 
2. When you had to use the command `rm`. 
3. When you had to use the command `cp`. 
