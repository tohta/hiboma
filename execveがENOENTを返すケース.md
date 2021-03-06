# execve(2) で ENOENT

 * http://archive.linux.or.jp/JF/JFdocs/kernel-docs-2.6/binfmt_misc.txt.html

ローダが無い場合に do_execve -> search_binary_handler で ENOENT 返すケースがある

```c
/*
 * cycle the list of binary formats handler, until one recognizes the image
 */
int search_binary_handler(struct linux_binprm *bprm,struct pt_regs *regs)
{
	unsigned int depth = bprm->recursion_depth;
	int try,retval;
	struct linux_binfmt *fmt;

	retval = security_bprm_check(bprm);
	if (retval)
		return retval;
	retval = ima_bprm_check(bprm);
	if (retval)
		return retval;

	retval = audit_bprm(bprm);
	if (retval)
		return retval;

    // ここで ENOTENT
	retval = -ENOENT;
	for (try=0; try<2; try++) {
		read_lock(&binfmt_lock);
        // ローダーをイテレート
		list_for_each_entry(fmt, &formats, lh) {
			int (*fn)(struct linux_binprm *, struct pt_regs *) = fmt->load_binary;
			if (!fn)
				continue;
			if (!try_module_get(fmt->module))
				continue;
			read_unlock(&binfmt_lock);
			retval = fn(bprm, regs);
			/*
			 * Restore the depth counter to its starting value
			 * in this call, so we don't have to rely on every
			 * load_binary function to restore it on return.
			 */
			bprm->recursion_depth = depth;
			if (retval >= 0) {
				if (depth == 0)
					tracehook_report_exec(fmt, bprm, regs);
				put_binfmt(fmt);
				allow_write_access(bprm->file);
				if (bprm->file)
					fput(bprm->file);
				bprm->file = NULL;
				current->did_exec = 1;
				proc_exec_connector(current);
				return retval;
			}
			read_lock(&binfmt_lock);
			put_binfmt(fmt);
			if (retval != -ENOEXEC || bprm->mm == NULL)
				break;
			if (!bprm->file) {
				read_unlock(&binfmt_lock);
				return retval;
			}
		}
		read_unlock(&binfmt_lock);
		if (retval != -ENOEXEC || bprm->mm == NULL) {
			break;
#ifdef CONFIG_MODULES
		} else {
#define printable(c) (((c)=='\t') || ((c)=='\n') || (0x20<=(c) && (c)<=0x7e))
			if (printable(bprm->buf[0]) &&
			    printable(bprm->buf[1]) &&
			    printable(bprm->buf[2]) &&
			    printable(bprm->buf[3]))
				break; /* -ENOEXEC */
			request_module("binfmt-%04x", *(unsigned short *)(&bprm->buf[2]));
#endif
		}
	}
	return retval;
}

EXPORT_SYMBOL(search_binary_handler);
```

## 該当バイナリの readelf の 結果

```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x000000000000001a 0x000000000000001a  R      1
      [Requesting program interpreter: /lib/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x000000000001db4c 0x000000000001db4c  R E    200000
  LOAD           0x000000000001e228 0x000000000021e228 0x000000000021e228
                 0x0000000000001468 0x00000000000018e0  RW     200000
  DYNAMIC        0x000000000001e6d8 0x000000000021e6d8 0x000000000021e6d8
                 0x00000000000001f0 0x00000000000001f0  RW     8
  NOTE           0x0000000000000254 0x0000000000000254 0x0000000000000254
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x000000000001c450 0x000000000001c450 0x000000000001c450
                 0x000000000000038c 0x000000000000038c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     8
  GNU_RELRO      0x000000000001e228 0x000000000021e228 0x000000000021e228
                 0x0000000000000dd8 0x0000000000000dd8  R      1
```

## IRC

```
02/05 13:51:49 mizzy: nscdのソースいじったりしてみよう、と思って、glibcのソース持ってきてコンパイルしたnscd動かそうとしたら
02/05 13:51:58 mizzy: % ./nscd/nscd
02/05 13:52:00 mizzy: zsh: no such file or directory: ./nscd/nscd
02/05 13:52:05 mizzy: というなんだこれ案件。
02/05 13:52:17 mizzy: ファイルは確かに存在してるんだけど…
02/05 13:52:36 hiroyan: お なんすかね w
02/05 13:53:09 mizzy: straceとかしてみたけどよくわからない…
02/05 13:55:22 mizzy: [mizzy@users201]~/src/build-2.12.1% file ./nscd/nscd
02/05 13:55:24 mizzy: ./nscd/nscd: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.4.0, not stripped
02/05 13:55:27 mizzy: ファイルはあるんだよねー
02/05 13:56:32 mizzy: lddしてみても問題なさそう
02/05 13:59:44 mizzy: configureオプションまわり見てみよう
02/05 14:01:10 hiroyan: http://stackoverflow.com/questions/5234088/execve-file-not-found-when-stracing-the-very-same-file 同じハマり方の人がいました
02/05 14:01:14 mizzy: お
02/05 14:03:23 hiroyan: The file you're trying to execute (…/lmutil) exists but its “loader” doesn't exist, where
02/05 14:03:33 hiroyan: loader は ld の事すよね
02/05 14:03:51 mizzy: かな
02/05 14:05:00 hiroyan: nscd の ldd の結果いただけますか
02/05 14:05:09 hiroyan: てか自分もログインしたらいいすね
02/05 14:05:22 hiroyan: ああ 鍵が ..
02/05 14:06:19 mizzy: お、鍵追加しとくよ
02/05 14:06:51 hiroyan: すません
02/05 14:06:53 hiroyan: ssh-rsa ***
02/05 14:07:14 mizzy: うす、ちょっと待ってねー
02/05 14:09:31 mizzy: ひろやん、設定してみましたー
02/05 14:09:38 hiroyan: azms !
02/05 14:10:03 mizzy: ~mizzy/srcの下にglibcのソースあるので、適当にさわるなり持っていくなりしてください
02/05 14:10:23 hiroyan: ログインできました。ありがとうございますー!1
02/05 14:10:28 mizzy: よかった
02/05 14:11:06 mizzy: glibcをソース展開したディレクトリでビルドしようとすると、別ディレクトリでやれ、って怒られる
02/05 14:14:41 hiroyan: readelf -a ~mizzy/src/build-2.12.1/nscd/nscd | less で
02/05 14:14:58 hiroyan: 中味みてたら
02/05 14:14:59 hiroyan:      [Requesting program interpreter: /lib/ld-linux-x86-64.so.2]
02/05 14:15:18 hiroyan: /lib64 じゃなくて /lib になってたので
02/05 14:15:25 mizzy: お
02/05 14:15:30 hiroyan: sudo ln -sv /lib64/ld-linux-x86-64.so.2 /lib
02/05 14:15:43 hiroyan: 試しにリンクはってみたら 挙動変わった気がします
02/05 14:15:55 mizzy: おお
02/05 14:15:59 hiroyan: 解決策としては間違ってると思うんですが w
02/05 14:16:06 hiroyan: /lib64 に向くようにする方法が。
02/05 14:17:13 mizzy: lddすると /lib/ld-linux-x86-64.so.2 => /lib64/ld-linux-x86-64.so.2 (0x0000003000000000) って出てたから、勝手に辿ってくれてると思ったんだけど、そういうものでもないのか…
02/05 14:17:51 hiroyan: そこ謎ですね
02/05 14:18:00 mizzy: とりあえずちゃんと起動して動いた！
02/05 14:18:15 mizzy: デバッグしたいだけなので、この対応で十分です。助かりました−、ありがとう！
02/05 14:18:26 hiroyan: 了解しましたー
```

ローダが /lib64 に向いてるようにコンパルすんのってどうすんの?

