---
title: 'How I manage my work like a boss'
date: 2022-10-29
draft: false
tags: [ "planning", "management", "productivity" ]
---

If you have deadlines on top of people asking you to do stuff ASAP, on top of issues that need to be fixed immediately, on top of many meetings throughout the day, it's easy to lose track of your to do list or prioritise the wrong things.

My to do list method has helped me keep an overview of everything I need to do. It's low overhead once you get used to it.
It's flexible enough to acommodate unexpected work coming up throughout the day or the week, whilst being rigorous enough to keep track of deadlines and make sure everything I ought to be doing gets done.

I use an app called [Obsidian](https://obsidian.md) to manage my notes, but this method will probably work with many other note taking apps. Obsidian uses Markdown for formatting which I find very useful.

# Daily note plugin

I use the daily note feature of Obsidian to easily create a fresh note everyday.
You can enable it by clicking on the settings icon (looks like a gear) on the bottom left, going to 'Core plugins' and enabling 'Daily notes'.
Once enabled, you can edit the settings of the daily note by clicking on the gear next to the plugin name in the same settings tab.

I recommend having a separate folder just for daily notes, you can specify the folder name in 'New file location'. I call mine 'Logs', as my notes also function as daily logs of my work.
You can provide a template file for your notes as well, which I will get in to later, and you can choose to open the daily note when you open Obsidian.
If you choose not to open your daily note on startup, you can press the daily note button on the left of your Obsidian window (it looks like a calendar) to open/create the note.

# The daily note format

My daily notes consist of two bullet lists. At the top, I have the list of tasks I have completed that day. Then I have a heading called 'TODO' followed by another bullet list of *all* my to do's.
Those are not just the to do's of the day, but the longer term ones as well.

I'll show you an example. Markdown uses `-` at the beginning of a line for bullet lists and `#` for headings. You can use `[[...]]` to link to other notes in your Obsidian vault.

```
- Task that I have completed
- Another tasks that I have completed
- Meeting about xxx. [[Meeting notes]]
- A task that I have worked on but not completed

# TODO
- Task that I have to do urgently
- Another urgent task

- Task needs to be finished this sprint
- Another task for this sprint

- 29-10-2022 Task with deadline.
  - Subtask
  - Other subtask
- 01-2023 Other task with deadline. [[Task details]]
```

# Creating the TODO list

Every working day, I copy and paste the whole TODO list from the previous daily note to the current one.

As I go through emails and messages in the morning, I'll add every request to the top of the list. I'll also add meetings I have that day.

Then I'll reorder the list to create a rough order of the things I want to do that day. Anything I don't expect to work on that day, I'll leave down lower on the list, separated by an empty line.
Anything that needs to be done longer term, say, in more than a weeks time, I'll leave down even lower than that, also separated by an empty line.

# Managing sudden urgent tasks

When something urgent comes up during the day, I'll simply add it to the top or near the top of the TODO list. Any tasks that I can't complete that day because of it will just move onto the next day when I copy paste the TODO list.

# Managing deadlines

Any task on the TODO list that has a deadline I'll prefix with the date it needs to be completed. That way I see all my deadlines every morning.
I'll update the dates on the daily list whenever a deadline changes.
As a deadline comes closer, I'll move the task higher up the TODO list.

# The log of completed work

Whenever I complete a task, I'll move the task out of the TODO area into the bullet list at the top of the daily note. I'll call that list the 'log'.
If I needed to do anything special or unexpected to complete the task, I add that to the bullet point.

When you do this, you'll end up with logs of what you did every day. This could come in handy if at any point you need to find out what you were doing at a certain time, or when you were doing something, using the search box of Obsidian. You'll have answers if stakeholders ask you why a certain thing couldn't be finished at a certain time.

If you need to keep track of hours worked on certain tasks, you can also add the time of completion to each bullet point.

# Longer term tasks

If you work on a task but don't complete it that day, just add a bullet point of what you did to the log, without removing the task from the TODO list.

If you want, you can also split up tasks in the TODO list by adding bullet points below it with extra indentation.

# Linking to other notes

Aside from daily notes, I also use Obsidian to make more detailed notes about other things. For example, I'll create a note for each meeting, and I'll create a note if I'm doing research for a certain feature.

Obsidian allows you to link notes to other notes using `[[...]]` notation:with. So if I have a meeting in the daily note, I'll link it to the meeting notes, and if I have notes related to a certain task, I'll link the bullet point to the task.

You can of course also link externally using URL's. For example, you might want to link to a ticket on a planner board.

# Organising many daily notes

Over time, you'll have a long long list of daily notes. You can easily organise them per month.
When a new month has begun, simply create a folder with the year and month, e.g. `2022-10`, and move all the notes from that period into it.
That makes it really easy to look up things you did during a certain time period.

# Conclusion

This way of organising tasks has helped me stay on top of things, long term and short term, whilst staying flexible enough to react to things that need immediate attention.
It has also allowed me to keep a detailed log of everything I have done, and when I have done it.
When communicating with stakeholders, I'll know what I was doing at a certain time, and why things might have taken longer.
But most of the time that isn't necessary, because I have an overview of everything I need to do and can make the right decisions to get everything done on time.

