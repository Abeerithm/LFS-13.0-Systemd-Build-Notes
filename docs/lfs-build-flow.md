<div dir="rtl">

# مخطط مراحل بناء LFS

هذا المخطط يوضح المسار العام لبناء نظام Linux From Scratch 13.0-systemd، من تجهيز النظام المضيف حتى الوصول إلى النظام النهائي القابل للإقلاع.

الهدف من المخطط هو إعطاء صورة ذهنية واضحة لتسلسل البناء، وليس استبدال خطوات الكتاب الرسمية أو شرح كل حزمة بالتفصيل.

</div>

```mermaid
flowchart TD
    A[Host System Preparation<br/>Debian on VirtualBox] --> B[Disk Partitioning<br/>Create and mount /mnt/lfs]
    B --> C[Download Sources<br/>Store packages in /sources]
    C --> D[Temporary Cross-Toolchain<br/>Binutils Pass 1, GCC Pass 1, Glibc, Libstdc++]
    D --> E[Temporary Tools<br/>M4, Ncurses, Bash, Coreutils, Make, Sed, Tar, Xz...]
    E --> F[Prepare Chroot<br/>Change ownership and mount virtual filesystems]
    F --> G[Enter Chroot Environment]
    G --> H[Temporary Tools inside Chroot<br/>Gettext, Bison, Perl, Python, Texinfo, Util-linux]
    H --> I[Clean Temporary System<br/>Remove /tools and temporary files]
    I --> J[Final System Packages<br/>Chapter 8]
    J --> K[System Configuration<br/>Chapter 9]
    K --> L[Kernel and Boot Setup<br/>Chapter 10]
    L --> M[First Boot<br/>LFS System]
```

<div dir="rtl">

## ملاحظات على المخطط

* هذا المخطط يوضح الصورة العامة لمراحل بناء LFS.
* تفاصيل كل مرحلة موثقة في ملفات الفصول داخل مجلد `docs/`.
* المرحلة الحالية من المشروع تقع ضمن بناء حزم النظام النهائي في Chapter 8.
* تم استخدام اللغة الإنجليزية داخل المخطط لأن Mermaid يتعامل معها بشكل أوضح من النص العربي المختلط.

</div>
