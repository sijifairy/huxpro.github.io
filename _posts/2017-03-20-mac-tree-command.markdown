---
layout:     post
title:      "Mac显示目录结构Tree命令"
subtitle:   "Activity Startup Re-arrangement on Post-Nougat Devices"
date:       2017-03-20 00:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
---

mac下默认没有tree命令，需要安装：

```shell
brew install tree
```

[tree -L 1]   指定层级

```text
.
├── Applications
├── Applications\ (Parallels)
├── Desktop
├── Documents
├── Downloads
├── Library
├── Movies
├── Music
├── Pictures
├── Public
├── Tmp
├── Workspace
├── genymotion-log.zip
├── keystore.jks
├── myblog
└── npm-debug.log
```

[tree -C -L 1]  在文件和目录清单加上色彩，便于区分各种类型。

[tree -C -D -L 1]   列出文件或目录的更改时间。

```text
.
├── [Feb  6 17:08]  Applications
├── [Mar 14  2016]  Applications\ (Parallels)
├── [Mar 20 19:14]  Desktop
├── [Mar 15 11:05]  Documents
├── [Mar 20 17:53]  Downloads
├── [Mar  8 11:42]  Library
├── [Nov  6 11:05]  Movies
├── [Mar 29  2015]  Music
├── [Jul 26  2016]  Pictures
├── [Mar 15  2015]  Public
├── [Feb 18 10:06]  Tmp
├── [Mar 20 20:30]  Workspace
├── [Mar 10 17:55]  genymotion-log.zip
├── [Mar 30  2016]  keystore.jks
├── [Feb 20 17:42]  myblog
└── [Feb  9 19:37]  npm-debug.log
```

[tree -C -D -f -L 1]    在每个文件或目录之前，显示完整的相对路径名称。

```text
.
├── [Feb  6 17:08]  ./Applications
├── [Mar 14  2016]  ./Applications\ (Parallels)
├── [Mar 20 19:14]  ./Desktop
├── [Mar 15 11:05]  ./Documents
├── [Mar 20 17:53]  ./Downloads
├── [Mar  8 11:42]  ./Library
├── [Nov  6 11:05]  ./Movies
├── [Mar 29  2015]  ./Music
├── [Jul 26  2016]  ./Pictures
├── [Mar 15  2015]  ./Public
├── [Feb 18 10:06]  ./Tmp
├── [Mar 20 20:30]  ./Workspace
├── [Mar 10 17:55]  ./genymotion-log.zip
├── [Mar 30  2016]  ./keystore.jks
├── [Feb 20 17:42]  ./myblog
└── [Feb  9 19:37]  ./npm-debug.log
```

[tree -C -D -p -f -L 1]    列出权限标示。

```text
.
├── [drwxr-xr-x Feb  6 17:08]  ./Applications
├── [drwxr-xr-x Mar 14  2016]  ./Applications\ (Parallels)
├── [drwx------ Mar 20 19:14]  ./Desktop
├── [drwx------ Mar 15 11:05]  ./Documents
├── [drwx------ Mar 20 17:53]  ./Downloads
├── [drwx------ Mar  8 11:42]  ./Library
├── [drwx------ Nov  6 11:05]  ./Movies
├── [drwx------ Mar 29  2015]  ./Music
├── [drwx------ Jul 26  2016]  ./Pictures
├── [drwxr-xr-x Mar 15  2015]  ./Public
├── [drwxr-xr-x Feb 18 10:06]  ./Tmp
├── [drwxr-xr-x Mar 20 20:30]  ./Workspace
├── [-rw-r--r-- Mar 10 17:55]  ./genymotion-log.zip
├── [-rw-r--r-- Mar 30  2016]  ./keystore.jks
├── [drwxr-xr-x Feb 20 17:42]  ./myblog
└── [-rw-r--r-- Feb  9 19:37]  ./npm-debug.log
```

附上tree命令的完整说明：[tree --help]
```text
usage: tree [-acdfghilnpqrstuvxACDFJQNSUX] [-H baseHREF] [-T title ]
	[-L level [-R]] [-P pattern] [-I pattern] [-o filename] [--version]
	[--help] [--inodes] [--device] [--noreport] [--nolinks] [--dirsfirst]
	[--charset charset] [--filelimit[=]#] [--si] [--timefmt[=]<f>]
	[--sort[=]<name>] [--matchdirs] [--ignore-case] [--] [<directory list>]
  ------- Listing options -------
  -a            All files are listed.
  -d            List directories only.
  -l            Follow symbolic links like directories.
  -f            Print the full path prefix for each file.
  -x            Stay on current filesystem only.
  -L level      Descend only level directories deep.
  -R            Rerun tree when max dir level reached.
  -P pattern    List only those files that match the pattern given.
  -I pattern    Do not list files that match the given pattern.
  --ignore-case Ignore case when pattern matching.
  --matchdirs   Include directory names in -P pattern matching.
  --noreport    Turn off file/directory count at end of tree listing.
  --charset X   Use charset X for terminal/HTML and indentation line output.
  --filelimit # Do not descend dirs with more than # files in them.
  --timefmt <f> Print and format time according to the format <f>.
  -o filename   Output to file instead of stdout.
  -------- File options ---------
  -q            Print non-printable characters as '?'.
  -N            Print non-printable characters as is.
  -Q            Quote filenames with double quotes.
  -p            Print the protections for each file.
  -u            Displays file owner or UID number.
  -g            Displays file group owner or GID number.
  -s            Print the size in bytes of each file.
  -h            Print the size in a more human readable way.
  --si          Like -h, but use in SI units (powers of 1000).
  -D            Print the date of last modification or (-c) status change.
  -F            Appends '/', '=', '*', '@', '|' or '>' as per ls -F.
  --inodes      Print inode number of each file.
  --device      Print device ID number to which each file belongs.
  ------- Sorting options -------
  -v            Sort files alphanumerically by version.
  -t            Sort files by last modification time.
  -c            Sort files by last status change time.
  -U            Leave files unsorted.
  -r            Reverse the order of the sort.
  --dirsfirst   List directories before files (-U disables).
  --sort X      Select sort: name,version,size,mtime,ctime.
  ------- Graphics options ------
  -i            Don't print indentation lines.
  -A            Print ANSI lines graphic indentation lines.
  -S            Print with CP437 (console) graphics indentation lines.
  -n            Turn colorization off always (-C overrides).
  -C            Turn colorization on always.
  ------- XML/HTML/JSON options -------
  -X            Prints out an XML representation of the tree.
  -J            Prints out an JSON representation of the tree.
  -H baseHREF   Prints out HTML format with baseHREF as top directory.
  -T string     Replace the default HTML title and H1 header with string.
  --nolinks     Turn off hyperlinks in HTML output.
  ---- Miscellaneous options ----
  --version     Print version and exit.
  --help        Print usage and this help message and exit.
  --            Options processing terminator.
```
