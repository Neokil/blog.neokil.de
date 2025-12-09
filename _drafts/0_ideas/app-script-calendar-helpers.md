---
layout: post
title: "Apps-Script Calendar Helpers"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general apps-script automation calendar
---

I just recently found out about googles "Apps Script" (available at script.google.com).
It allows you to write automation scripts that can be triggered on a schedule or on an event that run within the google workspace.
I find it really useful to add some automation to my calendar but it can also integrate other google products or external apis.
I currently use it for two usecases:
1. Add blockers for private appointments: I have a private calendar that I import from my private email that is only visible to me. And I want to have blockers in my company calendar that block 15 minutes before and after my private appointments. So for example if I have an appointment at the doctor I have a few minutes before that to finish my work and get going. And when coming back I don't have to hurry as the calendar is still blocked for a few more minutes.
2. Add "meeting preparation blockers" before every non-recurring meeting. I don't like to go into meeting unprepared as I think that makes them hightly inefficient, but when you are jumping between meeting thats not always possible and you are still at the old meeting with your mind. For that I automatically add a 15 minute preparation blocker before every meeting so I can already put my mind on the topic, maybe do some quick research and then when the meeting starts it will be much more efficient.

Solving those two problems was really easy with app script and in the following I will walk you through the steps to do it.
1. go to `https://script.google.com/home`
2. create a new project, this will open a new code-space
3. On the image img/posts/app-script-calendar-helpers-1.png I marked the top bar where you can edit the project-name, the left bar where you can find the different sections like the Editor and the Triggers and the Main Code-Window with save and run/debug controls.
4. Now that we have the project created we can add the code. It is written in javascript and you are creating functions and later when defining the triggers you can select which function to execute. For the source-code you can have a below, where I provide the code that I use to solve the problems described above.
5. The intellisense and ability to always press "Run" or "Debug" and see the "console.log" output makes it really easy to develop scripts. Be aware that those scripts can directly modifiy your calendar, so be careful not to flood it with new events.
6. When the code is ready and working we need to define a trigger so it runs automatically. For that we click on the stopwatch symbol in the left menu which opens the triggers section.
7. Then we click on "add trigger" in the bottom right and in the dialog we can then define which function to run, what type of trigger we want and some trigger options. Click on Save and then we have the automation running.

To have an overview over all automations that you have running you can go back to the home-page (https://script.google.com/home) and there you have "My Executions" and "My Triggers" that show what you have set up and what was running.

I really like this tool as it is giving me an easy way to automate my work environment. I am pretty sure I have not even scratched the surface of what is possible, so feel free to get creative and make your live a bit easier.

## Code-Snippets:
### Add Meeting Preparation Blockers:
```
const prepDurationMinutes = 15;
const checkDaysInAdvance = 7;

function createPrepBlocks() {
  const calendar = CalendarApp.getDefaultCalendar();
  const now = new Date();
  const future = new Date(now.getTime() + 1000 * 60 * 60 * 24 * checkDaysInAdvance);
  const events = calendar.getEvents(now, future);

  events.forEach(event => {
    if (event.getGuestList().length < 1) return;
    if (event.isAllDayEvent()) return;
    if (event.isRecurringEvent()) return;


    const start = event.getStartTime();
    const prepStart = new Date(start.getTime() - prepDurationMinutes * 60 * 1000);
    const prepEnd = start;

    // Check if prep event already exists
    const existing = calendar.getEvents(prepStart, prepEnd)
      .filter(e => e.getTitle() === "Meeting Preparation");

    if (existing.length === 0) {
      var prepEvent = calendar.createEvent("Meeting Preparation", prepStart, prepEnd);
      prepEvent.setDescription(event.getDescription())
    }
  });
}
```

### Add Blockers for Private Appointments
```
const blockerExtensionMinutes = 15;
const checkDaysInAdvance = 30;

function createBlockers() {
  const mainCalendar = CalendarApp.getDefaultCalendar();
  const privateCalendar = CalendarApp.getCalendarById("...");

  const privateEvents = privateCalendar.getEvents(new Date(), new Date((new Date()).getTime() + 1000 * 60 * 60 * 24 * checkDaysInAdvance))
  privateEvents.forEach(privateEvent => {
    if (privateEvent.isAllDayEvent()) return;
    if (privateEvent.isRecurringEvent()) return;

    const blockerStart = privateEvent.getStartTime()
    blockerStart.setMinutes(blockerStart.getMinutes() - blockerExtensionMinutes);
    
    const blockerEnd = privateEvent.getEndTime()
    blockerEnd.setMinutes(blockerEnd.getMinutes() + blockerExtensionMinutes);


    const existing = mainCalendar.getEvents(blockerStart, blockerEnd).filter(e => e.getTitle() === "Blocker");
    if (existing.length === 0) {
      console.info("creating blocker for " + privateEvent.getTitle());
      var blocker = mainCalendar.createEvent("Blocker", blockerStart, blockerEnd);
    }
  });
}

```