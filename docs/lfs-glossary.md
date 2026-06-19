# LFS Glossary Notes

This file contains important terms I encountered while studying and building Linux From Scratch 13.0-systemd.

The goal is not only to translate the terms, but also to explain how each concept appears during the actual LFS build process.

## Core Concepts

| Term | Arabic Meaning | Practical Meaning in LFS | Where It Appears |
|---|---|---|---|
| Toolchain | سلسلة أدوات البناء | مجموعة الأدوات الأساسية مثل GCC, Binutils, Glibc المستخدمة لبناء النظام | Chapters 5–8 |
| Cross-Compilation | الترجمة المتقاطعة | بناء برامج لنظام مختلف عن النظام المضيف مؤقتًا أثناء إعداد LFS | Temporary Toolchain |
| Chroot | تغيير جذر النظام | الدخول إلى بيئة LFS وكأنها نظام مستقل عن النظام المضيف | Chapter 7 |
| Host System | النظام المضيف | نظام Debian المستخدم لبناء LFS | Before chroot |
| Target System | النظام الهدف | نظام LFS الذي يتم بناؤه | Throughout the build |
| Build Directory | مجلد البناء | مجلد منفصل لتجميع الحزمة بدون تلويث ملفات المصدر | Most packages |
| Source Directory | مجلد المصدر | المجلد الذي يحتوي على كود الحزمة الأصلي | `/sources` |

## Build and Compilation Terms

| Term | Arabic Meaning | Practical Meaning in LFS | Example |
|---|---|---|---|
| Configure | تهيئة البناء | تجهيز الحزمة حسب المسارات والخيارات المطلوبة قبل التجميع | `./configure --prefix=/usr` |
| Make | بناء / تجميع | تنفيذ عملية compilation بناءً على Makefile | `make` |
| Make install | تثبيت | نسخ الملفات المبنية إلى أماكنها النهائية في النظام | `make install` |
| Test Suite | مجموعة الاختبارات | اختبارات تتحقق من صحة بناء الحزمة | `make check` |
| Expected Failure | فشل متوقع | اختبار معروف أنه قد يفشل ولا يعني بالضرورة وجود مشكلة | GCC tests |
| Sanity Check | فحص سلامة | اختبار سريع للتأكد أن الأداة تعمل بشكل صحيح | GCC sanity check |

## System and Filesystem Terms

| Term | Arabic Meaning | Practical Meaning in LFS | Example |
|---|---|---|---|
| Symlink | رابط رمزي | ملف يشير إلى ملف أو أمر آخر | `cc -> gcc` |
| Mount | ربط نظام ملفات | إتاحة قسم أو نظام ملفات داخل مسار معين | `/mnt/lfs` |
| Virtual Filesystems | أنظمة ملفات افتراضية | أنظمة مثل `/proc`, `/sys`, `/dev` يحتاجها chroot للتعامل مع النظام | Chapter 7 |
| Root Directory | جذر النظام | أعلى نقطة في بنية الملفات | `/` |
| Prefix | مسار التثبيت الأساسي | يحدد أين سيتم تثبيت ملفات الحزمة | `--prefix=/usr` |


