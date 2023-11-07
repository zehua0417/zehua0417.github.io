---
title: 个人bashscript脚本分享
date: 2023-11-7
update: 2023-11-7
tag: 
  - linux
  - shell
categories:
  - study
---

# 个人bashscript脚本分享

1. backup.sh

> 备份脚本

```shell
#!/bin/bash

# Parse command-line arguments
ARGS=$(getopt -o d:t:p:: --long dir:,tag:,passwd::,tool: -n "$0" -- "$@")
if [ $? != 0 ]; then
    echo "Backup Script Usage Guide

Overview:
  This script creates file backups, supporting various compression tools and password protection options. It's used for periodic backups of files or directories.

Usage:
  ./backup_script.sh [options] files/directories...

Options:
  -d, --dir Target Directory:
    Specify the target directory for the backup. Default is 'Default.'

  -t, --tag Backup Tag:
    Specify a backup tag to name the backup file. Default is 'anonymousfile.'

  -p, --passwd [password]:
    Enable password protection for the backup. Provide a password or leave it blank. If empty, you'll be prompted for a password.

  --tool Tool:
    Specify the compression tool: 'tar' or '7z.' Default is 'tar.' Note that 'tar' doesn't support password protection.

Files/Directories:
  Specify files or directories to back up.

Example Usage:
  ./backup_script.sh -d my_backup -t important_data -p mypassword ~/Documents
  Create a password-protected backup of '~/Documents' in the 'my_backup' directory with the tag 'important_data.'

  ./backup_script.sh --dir project_backup --tag code_backup ~/Projects
  Create an unprotected backup of '~/Projects' in the 'project_backup' directory with the tag 'code_backup.'

  ./backup_script.sh --tool 7z ~/Documents/file.txt ~/Desktop/pictures
  Create an unprotected backup using the 7z tool for 'file.txt' and the 'pictures' directory.

Notes:
  1. If no password is provided, you'll be prompted during the backup.
  2. Ensure the selected tool ('tar' or '7z') is installed.
  3. When enabling password protection with -p, '7z' is used.
  4. The default backup directory is '/f/Backups.' The directory specified after the -d option will be appended to '/f/Backups.' If you want to change the directory, you need to modify the script code.

Feedback and Support:
  For questions or assistance, contact the script author.

Caution:
  Use this script with care to safeguard your backup data."
    echo "Terminating..."
    exit 1
fi

eval set -- "$ARGS"

# Initialize variables with default values
dir="Default"
tag="anonymousfile"
usepasswd="false"
password=""
usedtool="tar"
file_list=()

while true; do
    case "$1" in
        -d|--dir)
            dir=$2
            shift 2
            ;;
        -t|--tag)
            tag=$2
            shift 2
            ;;
        -p|--passwd)
            case "$2" in
                "")
                    usepasswd="true"
                    shift 2
                    ;;
                *)
                    usepasswd="true"
                    password=$2
                    shift 2
                    ;;
            esac
            ;;
        --tool)
            usedtool=$2
            shift 2
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Internal error!"
            exit 1
            ;;
    esac
done

file_list=("$@")

# Check if the file list is empty
if [ ${#file_list[@]} -eq 0 ]; then
    echo "Error: File list is empty."
    exit 1
fi

# Define the backup directory
backup_dir="/f/Backups/$dir"

# Check if the backup directory exists
if [ ! -d "$backup_dir" ]; then
    read -p "Directory $dir does not exist. Create it? (y/n) " create_flag
    if [ "$create_flag" = "y" ]; then
        mkdir -p "$backup_dir"
    else
        echo "Did not create the backup directory."
        exit 1
    fi
fi

# Generate the backup filename
backup_filename="backup-$tag-$(date +'%Y%m%d%H%M%S')"

# Create a temporary directory for backup
backup_temp_dir="$backup_dir/$backup_filename"
mkdir -p "$backup_temp_dir"

# Copy files to the temporary directory
cp -r "${file_list[@]}" "$backup_temp_dir"
if [ $? -ne 0 ]; then
    echo "Error: Failed to copy files to the backup directory."
    exit 1
fi

# Create a compressed archive
echo "Start compress"
if [ "$usedtool" = "tar" ]; then
    if [ -n "$usepasswd" ]; then
        read -p "Note: Password protection is not available with 'tar.' Continue? (y/n) " continue_flag
        if [ "$continue_flag" != "y" ]; then
            echo "Removing temp dir"
            rm -rf "$backup_temp_dir"
            echo "You can set a password using '-s 7z'."
            exit 1
        fi
    fi
    tar -czf "$backup_dir/$backup_filename.tar.gz" -C "$backup_dir" "$backup_filename"
elif [ "$usedtool" = "7z" ]; then
    if [ "$usepasswd" = "false" ]; then
        7z a -bso0 -bsp0 "${backup_dir}/${backup_filename}.7z" -r -t7z "$backup_temp_dir"
    else
        if [ -n "$password" ]; then
            7z a -bso0 -bsp0 -p"$password" "${backup_dir}/${backup_filename}.7z" -r -t7z "$backup_temp_dir"
        else
            echo -n "Enter password: "
            read -s password
            echo
            7z a -bso0 -bsp0 -p"$password" "${backup_dir}/${backup_filename}.7z" -r -t7z "$backup_temp_dir"
        fi
    fi
else
    echo "Error: Unsupported compression tool: $usedtool"
    echo "Removing temp dir"
    rm -rf "$backup_temp_dir"
    exit 1
fi

# Remove the temporary backup directory
echo "Removing temp dir"
rm -rf "$backup_temp_dir"

# Check if the backup was successful
if [ $? -eq 0 ]; then
    case "$usedtool" in
        tar)
            echo "Backup successful: $backup_dir/$backup_filename.tar.gz"
            ;;
        7z)
            echo "Backup successful: $backup_dir/$backup_filename.7z"
            ;;
        *)
            echo "Unknown backup tool used."
            ;;
    esac
else
    echo "Backup failed. Something went wrong."
fi

```

2. dmd.sh

> md + latex -> pdf 自动检测编译脚本

```shell
#!/bin/bash

ARGS=$(getopt -o i:o:g:t: --long input:,output:,gaptime:,template:,toc -n "$0" -- "$@")
if [ $? != 0 ]; then
    echo "用法: $0 -i <input_md> -o <output_pdf>"
    exit 1
fi

eval set -- "$ARGS"

input_md=''
output_pdf=''
initial_hash=''
current_hash=''
tocflag="false"
gaptime=3  # 将等待时间设置为3秒
templatefile=/c/Users/admin/Documents/bashDoc/eisvogel.tex


while true; do
    case "$1" in
        -i|--input)
            input_md=$2
            if [ ! -f "$input_md" ]; then
                echo "找不到输入文件: $input_md"
                exit 1
            fi
            initial_hash=$(md5sum "$input_md")
            shift 2
            ;;
        -o|--output)
            output_pdf=$2
            shift 2
            ;;
        -g|--gaptime)
            gaptime=$2
            shift 2
            ;;
        -t|--template)
            echo "templatefile: $2"
            templatefile=$2
            shift 2
            ;;
        --toc)
            tocflag="true"
            shift 1
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "错误: 意外的参数 - $1"
            echo "用法: $0 -i <input_md> -o <output_pdf>"
            exit 1
            ;;
    esac
done


echo "input: $input_md"
echo "output: $output_pdf"
echo "template: $templatefile"
echo "toc: $tocflag"


# 无限循环，每隔3秒检测一次文件是否被更改
while true; do
    if [ ! -f "$input_md" ]; then
        echo "找不到输入文件: $input_md"
        exit 1
    fi

    # 获取当前的文件哈希值
    current_hash=$(md5sum "$input_md")
    # 检查当前哈希值是否与初始哈希值不同
    if [ "$current_hash" != "$initial_hash" ]; then
        echo "文件已被更改！"

        if [ "$tocflag" == "true" ]; then
            pandoc --toc --number-sections "$input_md" -o "$output_pdf" --from markdown --template "$templatefile" --filter pandoc-latex-environment --listings --pdf-engine=xelatex
        else
            pandoc "$input_md" -o "$output_pdf" --from markdown --template "$templatefile" --filter pandoc-latex-environment --listings --pdf-engine=exlatex
        fi
        pandoc_exit_code=$?

        if [ $pandoc_exit_code -eq 0 ]; then
            echo "pandoc运行成功！"
        else
            echo "pandoc运行失败 (退出状态码: $pandoc_exit_code)！"
        fi
        initial_hash="$current_hash"
    fi

    # 等待3秒
    sleep $gaptime
done

```