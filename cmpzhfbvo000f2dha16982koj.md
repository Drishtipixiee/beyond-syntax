---
title: "
What Actually Happens When You Press Enter on an INSERT Query"
datePublished: 2026-06-04T12:39:13.382Z
cuid: cmpzhfbvo000f2dha16982koj
slug: what-actually-happens-when-you-press-enter-on-an-insert-query

---

I'm writing SQL queries, getting the right output, sis happy, assignment submitted. Standard. And then at some point I just froze. Because I realized I had zero idea what was actually happening. Like, where is the data going? Not "the database" — I know it's the database, thanks. I mean actually. Physically. What is happening right now when I type INSERT and press enter?

I looked around the class. Nobody was asking this. I didn't ask either. But it sat in my brain like a splinter for two weeks until I went and figured it out myself. here's what I found and also why I think it's a little embarrassing that none of us asked sooner.

**What I thought was happening (spoiler: spreadsheet brain)**

My mental model was basically: database = big Excel sheet. Rows, columns, data sitting there neatly. You insert a row, it appears in the sheet. Done. Not wrong exactly. Just the kind of "not wrong" that becomes completely useless the moment anything breaks.

Real picture: databases store everything in fixed-size chunks called pages. typically 8KB (PostgreSQL) or 16KB (MySQL). Your entire database is just a massive pile of these pages. Every table you create lives inside them. When you save a row, the database finds a page with space, writes your data into it in a specific binary format, and updates internal bookkeeping to track where everything went. Already more interesting than the Excel mental image. And it gets weirder.

**The thing that actually broke my brain — it doesn't write to disk immediately**

When you run INSERT, your data does NOT go straight to the hard drive. It lands in something called a buffer pool first. Which is just RAM. Fast, temporary memory that the database keeps hot pages in because RAM is thousands of times faster than disk.

So you save something → it's in RAM → database marks that page as "dirty" (meaning: changes exist that haven't hit disk yet) → at some point in the background, it flushes to disk.

This is why databases are fast. And this is also why everyone gets nervous about crashes. If your machine dies before those dirty pages flush, that data is just gone. Which is obviously a problem. Which is why the next thing exists.

**The WAL — basically the database keeps a diary and that diary saves your life**

Before touching anything in the buffer pool, the database writes what it's about to do into a log...replays from the last checkpoint, figures out what completed and what didn't, and recovers accordingly. Write-Ahead Log. WAL. The name is exactly what it says — write to the log before you do the actual thing. Every INSERT, UPDATE, DELETE — log entry first, then execution. If the machine crashes mid-operation, the database replays the log on restart, figures out what completed and what didn't, and recovers accordingly.

This is how your data survives random power cuts. Not magic. Just a very disciplined logging system that someone designed specifically so you don't lose everything when your server decides to have a moment.

**Indexes — why your DBA gets that look every time someone mentions adding more indexes**

10 million rows. You search for one user by email. No index. The database reads every single page, every single row, top to bottom. Full table scan. As slow as it sounds. An index is a separate structure — usually a B-tree, which is a sorted tree that finds values in logarithmic time instead of linear. You create an index on the email column, the database builds and maintains this tree, now your search jumps straight to the right place instead of reading everything.

Tradeoff: indexes take space. Every write also has to update the index. More indexes = faster reads, slower writes. This is why you don't just index every column and call it a day. And this is why that one senior dev goes quiet every time someone casually says "let's just add more columns."

**So why does any of this matter**

Once I understood this, a bunch of stuff clicked that I'd honestly been faking for over a year.

Why does bulk inserting data crawl sometimes? Buffer pool pressure plus index updates on every single row. Why does adding an index on a huge table lock things up? It has to build the entire B-tree from scratch. Why does your database survive crashes but a plain text file doesn't? WAL. Why does everyone keep saying "disk I/O is the bottleneck"? Because the whole system is engineered around minimizing disk reads and the moment you break that assumption, everything slows down.

One level below what you're using. That's all it takes. You don't need to go all the way down to hardware. Just one level and suddenly your decisions start making actual sense instead of feeling like guesswork.

Also — every app you've ever used runs on some version of this exact system. Instagram. Swiggy. Your college ERP that crashes every single time during exam registration. Pages, WAL, B-trees, buffer pools. Same ideas, just at scale.Sit with that for a second.

**What I'd actually change about how this gets taught**

One lab. Just one. Where you deliberately crash a database mid-transaction, restart it, and watch it recover using the WAL. That demo alone would make this click for every student who's currently writing queries while having no idea what's happening underneath.

Instead we get "the database ACID properties" on a slide and the professor moves on like we all just nodded along and definitely didn't Google what ACID even stood for five minutes later.