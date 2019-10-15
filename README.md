# easy-copy


[github：easy-copy](https://github.com/boxker/easy-copy "github：easy-copy")

```python
import os
import sys
import time
import paramiko as pm

'''
host格式：
{
    "ip":"127.0.0.1",
    "port":22,
    "username":"root",
    "password":"123456",
    "file":"filename"
}
dst为host列表

要求：
源主机需要部署sshpass、scp
本机需要python3.4及以上，以及安装paramiko库
各源主机及目的主机能互联互通，ssh安全连接过
'''

# 默认运行配置：源
example_src = {
    "ip": "10.112.98.132",
    "port": 22,
    "username": "root",
    "password": "111",
    "file": "/root/111.o"
}

# 默认运行配置：目的，数组
example_dst = [
    {
        "ip": "10.112.98.134",
        "port": 22,
        "username": "root",
        "password": "222",
        "file": "/root/222.o"
    },
    {
        "ip": "10.112.98.151",
        "port": 22,
        "username": "root",
        "password": "333",
        "file": "/root/333.o"
    },
    {
        "ip": "10.112.96.67",
        "port": 22,
        "username": "root",
        "password": "444",
        "file": "/root/111.o"
    }
]

# 显示详细信息
# detail = False
detail = True


def do_copy(src_host, src_ssh, src_md5, dst_host, srcfile, compress=True, force=False, packname=".tarxjb"):
    print()
    print("*****cp*****")
    print()
    dst_ssh = create_ssh(dst_host)
    # dst_ssh.exec_command('ssh -o "StrictHostKeyChecking no" {name}@{ip}'.format(name=src_host["username"],ip=src_host["ip"]))
    # 检查对方是否已存在相同名字文件
    if check_exist(dst_ssh, dst_host["file"]):
        # 存在相同，判断是否强制覆盖，不覆盖则自动备份
        if detail:
            print("res:exist a same name file")
        if not force:
            if detail:
                print("res:do backup")
            backup_file_cmd = "mv {file} {file}.x.backup".format(
                file=dst_host["file"])
            dst_ssh.exec_command(backup_file_cmd)
    # 传输文件指令
    scp_cmd = "sshpass -p {pw} scp -P {port} {sfile} {duser}@{dip}:{dfile}".format(
        port=dst_host["port"], pw=dst_host["password"], sfile=srcfile+packname, duser=dst_host["username"], dip=dst_host["ip"], dfile=srcfile+packname)
    sin, sout, serr = src_ssh.exec_command(scp_cmd)
    if detail:
        print(sout.read().decode())
    else:
        sout.read().decode()
    if detail:
        print("res:scp done")
    dst_md5 = compu_md5(dst_ssh, srcfile+packname)
    if dst_md5 is None:
        # md5计算失败
        if detail:
            print("err:md5 failed")
        del_pack(dst_ssh, srcfile+packname)
        return False
    if src_md5 != dst_md5:
        # md5不同，需要撤回包
        if detail:
            print("err:md5 is not consist")
        del_pack(dst_ssh, srcfile+packname)
        return False
    # input("ddd")
    res = compu_tar(dst_ssh, file=srcfile, type="x",
                    compress=compress, packname=packname)
    # input("ddd")
    if detail:
        print("res:dst tar success")
    if not res:
        # 解压失败，需要撤回包
        if detail:
            print("err:tar failed")
        del_pack(dst_ssh, srcfile+packname)
        return False
    if detail:
        print("res:file check success")
    # 删除中间包
    # input("ddd")
    del_pack(dst_ssh, srcfile+packname)
    if detail:
        print("res:remove dst temp pack success")
    # 修改文件名
    # input("ddd")
    mv_file_cmd = "mv {file1} {file2}".format(
        file1=srcfile, file2=dst_host["file"])
    dst_ssh.exec_command(mv_file_cmd)
    if detail:
        print("res:filename update success")
    dst_ssh.close()
    if detail:
        print("ok: {file} done".format(file=dst_host["file"]))
    return True


def copy(src_host, dst, compress=True, force=False):
    # 复制文件，一对多
    # 获取于源ip的ssh链接
    src_ssh = create_ssh(src_host)
    # 确定压缩参数
    gzip = ""
    if compress:
        gzip = "z"
    # 判断文件是否存在
    res = check_exist(src_ssh, src_host["file"])
    if detail:
        print("src file exist? "+str(res))
    # 执行压缩、打包命令
    packname = "." + str(int(time.time())) + ".tarxjb"
    res = compu_tar(src_ssh, src_host["file"],
                    "c", compress=compress, packname=packname)
    if detail:
        print("res:tar success? "+str(res))
    # 计算原文件md5值
    src_md5 = compu_md5(src_ssh, src_host["file"]+packname)
    if detail:
        print("src md5 :"+str(src_md5))
    # 执行复制操作
    res_cp = []
    for row in dst:
        res_cp.append({row["file"]: do_copy(
            src_host, src_ssh, srcfile=src_host["file"], src_md5=src_md5, dst_host=row, packname=packname)})
    # 删除本地临时文件
    # input("ddd")
    del_pack(src_ssh, src_host["file"]+packname)
    if detail:
        print("res:remove src temp pack success")
    print()
    print("*****res*****")
    for index in res_cp:
        print(index)
    print("*****res*****")
    print()
    print("ok: all done!")
    src_ssh.close()


def del_pack(ssh, file):
    # 删除中间包
    del_cmd = "rm -f {file}".format(file=file)
    sin, sout, serr = ssh.exec_command(del_cmd)
    if detail:
        print("cmd: rm "+sout.read().decode())


def create_ssh(host):
    # 创建ssh链接
    ssh = pm.SSHClient()
    ssh.set_missing_host_key_policy(pm.AutoAddPolicy())
    ssh.connect(hostname=host["ip"], port=host["port"],
                username=host["username"], password=host["password"])
    return ssh


def compu_tar(ssh, file, type="c", compress=True, packname=".tarxjb"):
    # 打包及解包
    # gzip表示是否要用gzip压缩，默认压缩
    gzip = ""
    if compress:
        gzip = "z"
    # 打包指令
    tar_cmd = "tar c{gzip}vfpP {file}{packname} {file} 2>/dev/null".format(
        gzip=gzip, file=file, packname=packname)
    # 解包指令
    if type == "x":
        tar_cmd = "tar x{gzip}vfpP {file}{packname} 2>/dev/null".format(
            gzip=gzip, file=file, packname=packname)
    if detail:
        print("cmd: tar: " + tar_cmd)
    stdin, stdout, stderr = ssh.exec_command(tar_cmd)
    res = stdout.read().decode()
    if res == "":
        if detail:
            print("err: {file}{packname} tar failed".format(
                file=file, packname=packname))
        return False
    return True


def check_exist(ssh, file):
    # 检查文件是否存在
    exist_cmd = "find {file} 2>/dev/null".format(file=file)
    stdin, stdout, stderr = ssh.exec_command(exist_cmd)
    res = stdout.read().decode()
    if res == "":
        if detail:
            print("res: {file} is not exist".format(file=file))
        return False
    else:
        if detail:
            print("res: {file} is exist".format(file=file))
    return True


def compu_md5(ssh, file):
    # 计算文件md5
    src_md5_cmd = "md5sum {file} 2>/dev/null | awk \'{{print $1}}\'".format(
        file=file)
    stdin, stdout, stderr = ssh.exec_command(src_md5_cmd)
    src_md5 = stdout.read().decode()
    if detail:
        print("cmd: md5 :"+src_md5_cmd)
        print("res: cmd md5 res : "+src_md5)
    if src_md5 == "":
        if detail:
            print("err: {file} md5 or tar failed".format(file=file))
        src_md5 = None
    else:
        if detail:
            print("res: {file} md5 : {md5}".format(file=file, md5=src_md5))
    return src_md5


if __name__ == "__main__":
    print("easy tar tool")
    copy(example_src, example_dst, compress=True)

```
