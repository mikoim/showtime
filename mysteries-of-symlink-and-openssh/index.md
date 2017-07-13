<!-- $theme: gaia -->

勘違いから学ぶ symlink の謎 (とOpenSSH)
===

# ![](images/symlink.png)

##### Eshin Kunishima ( [@mikoim](https://github.com/mikoim) )

---

# OpenSSHで(昨日)発生した問題

OpenSSHで公開鍵認証を行うことが出来ない…

- Authentication refused: bad ownership or modes for directory **/pool/secure/ssh**
- SELINUX=**disabled**
- /home/foobar/.ssh (symlink) → /pool/secure/ssh
  - /home/foobar/.ssh: lrwxrwxrwx.．
  - /pool/secure/ssh: **drwx------.**
  - /pool/secure/ssh/*: **-rw-------.**

---

# `ls -l`の表示例

- 一般的(?)なファイル (SELinuxコンテキストラベル含)
  - Owner **RW**, Group **None**, Other **None** 

```
-rw-------. 1 ek ek  381 Oct 21  2016 mikoim.pub
```

- symlink
  - Owner **RWX**, Group **RWX**, Other **RWX** (← WTF!?)

```
lrwxrwxrwx.  1 ek   ek     28 Mar  7 19:59 .ssh -> /xxx
```

- setuid, setgid and sticky bits (← 省略)

---

# 私の推測(誤り)

- 当該symlinkのアクセス権限を見てun-secureだと判断しているのではないか…


※OpenSSHのソースコードと`man 2 stat`に答えが…

---

# 正解(symlink)

- ~~当該symlinkのアクセス権限を見てun-secureだと判断しているのではないか…~~
- OpenSSHは~/.ssh等の確認にstat(2)を使用する
- stat(2)は，symlinkを辿りリンク先のファイルについての状態を返す
  - symlinkの状態を取得したい時は，lstat(2)，fstatat(2)のflagにAT_SYMLINK_NOFOLLOWを渡す．
- Linuxではsymlinkのパーミッションを変更するsystem callは存在しない(昨日調べ)．


---

# 正解(OpenSSH)

OpenSSHは~/.sshからホームディレクトリもしくは/に至るまでのディレクトリを確認の対象としている．

```
ex) general environment
/home/foobar/.ssh (raise error if group or other can write)
/home/foobar (same)
break!

ex) special environment (like me)
/home/foobar/.ssh -> /pool/secure/ssh
/pool/secure/ssh (raise error if group or other can write)
/pool/secure (same)
/pool (same)
/ (same)
break!
```

---

# 正解(OpenSSH)

https://github.com/openssh/openssh-portable/blob/c9cdef35524bd59007e17d5bd2502dade69e2dfb/auth.c#L542

```c
for (;;) {
/* 省略 */
  if (comparehome && strcmp(homedir, buf) == 0)
    break;

  if ((strcmp("/", buf) == 0) || (strcmp(".", buf) == 0))
    break;
}
```

---

# ところで…

#### Linuxではsymlinkのパーミッションを変更するsystem callは存在しない(昨日調べ)．

## 他の*nixならあるのか?

---

# あります

- FreeBSD
  - symlink(2)で作成してからのlchmod(2)で変更可能
  - ただし，セットされた値はアクセス制御に使用されない
  - owner, groupの変更も可能

- Linux (比較用)
  - 変更不可
  - owner, groupは変更可能

---

# ちなみに

The Open Group Base Specifications Issue 7では…

The values of the file mode bits for the created symbolic link are unspecified. All interfaces specified by POSIX.1-2008 shall behave as if the contents of symbolic links can always be read, except that the value of the file mode bits returned in the st_mode field of the stat structure is unspecified.

## unspecified!!!

---

# 詳細は…

3割くらい(?)自己解決したので書きました（白目）

https://unix.stackexchange.com/questions/374084/openssh-refused-ssh-directory-with-a-symbolic-link

---

# ありがとうございました

---