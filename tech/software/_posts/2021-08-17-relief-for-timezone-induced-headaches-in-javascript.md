---
layout: page
title: Relief for Timezone-Induced Headaches in JavaScript
author: Marco Yuen
---

Whether you believe in a flat, globe, or another shape of earth, you will likely encounter the concept of time and time zones in some fashion as a developer. This post briefly explains and discusses how to make life easier when working with time zones in JavaScript by using Moment.js.

<!--more-->


### When to use time zones

If an app has users spread around the world or country and involves comparable timestamps, the app will need to keep track of time zones in these timestamps. So, how could we go about achieving this? You probably know that vanilla JavaScript comes bundled with Date objects. They can represent simple dates and times and are constructed by calling new Date(), which returns a date object representing the current date and time. Some Date methods include getters and setters for the year, month, day, hour, minute, and second values. The object’s time zone offset is based on the locale of the client browser. 

But where can we set this time zone? Well, we can’t in a simple way. In the case where time zone modification is only needed for display, I found that we could manually change the time immediately before rendering to adjust for the offset by calling the Date constructor with a time value that has removed the locale offset and added the desired offset. 

```javascript
const dateInDesiredTimezone = new Date(
  date.getTime() 
  + (date.getTimezoneOffset() 
    + <desired timezone offset in minutes>
  ) * 60 * 1000
);
```
We would then have a time value as if the Date object were created in a different time zone; however, the object’s timezone offset property still represents the locale time zone. This approach “works”, but it’s quite disgusting and should be avoided because the time object is no longer an accurate representation of the actual time. 

As you might imagine, if multiple successive time zone conversions are needed, further manipulation of the Date object will quickly result in painful headaches as we end up with unexplainable date/time values by losing track of the time zone that the object attempts to represent. Let’s instead use an external library that can help us deal with time zones.

### Introducing Moment.js

Moment.js is a library that provides a comprehensive toolbox for working with dates in JavaScript. More specifically, it provides a straightforward API for parsing, validating, manipulating, and displaying dates and times. This library becomes even more powerful when combined with the moment-timezone library, which will help us with our time zone troubles.

To install moment and moment-timezone:

If using npm:
```bash
npm install moment moment-timezone
```

If using yarn:
```bash
yarn add moment moment-timezone
```
To import in JS files with ES6 syntax:
```javascript
import moment from 'moment-timezone';
```

### Some available methods

Similar to the constructor for Date objects, calling moment() returns a moment object representing the current date and time. Date objects and date strings can also be parsed if passed in as an argument to the constructor. Along with getters/setters similar to those of Date objects, moment objects also have self-explanatory manipulation methods such as startOf(), add(), and customizable string formatting with format().

```javascript
const now = moment(); // current time and day
const beginningOfDay = moment().startOf('day'); // 12am midnight today
const noon = beginningOfDay.clone().add(12, 'h'); // 12pm noon today
```

So far, the time zone is set to the locale of the client browser (PST in my case). With moment-timezone, we also get access to a tz() method which allows us to easily switch between time zones by passing in a region name.

```javascript
const timeInSydney = noon.tz('Australia/Sydney'); // 5am on the day after (assuming the noon object was in PST) 
```

Neat! Now, we have a moment object set in the AEST time zone. It should be noted that moment objects are mutable, meaning that the noon variable is now also in AEST. Either using the clone() method or calling moment() with a moment object as an argument yields a copy of the original moment object.

To get the time as a string:

```javascript
const timeInSydneyString = timeInSydney.format('h:mm a z'); // ‘5:00 am AEST’
```

I’ve only included a minuscule subset of the available methods here in order to focus on time zone functionality. The rest of them can be found in the Moment.js documentation, linked at the bottom of this post.

### Considerations needed

Wherever possible, the date/time logic and storage should be implemented in a consistent time zone such as UTC. The time zone can be stored and passed around as a separate variable, allowing it to be used in the tz() method before rendering. Speaking from experience, if dates are stored and circulated in their own time zones, the code needed to handle the time zone conversions can quickly turn into spaghetti and introduce countless bugs as the app progressively gets more complex, which would bring us back to square one.

Using Moment.js and React, I noted that it was very straightforward to create a bare-bones timezone converter component while following the above considerations:

```tsx
import React, { useState } from 'react';
import moment, { Moment } from 'moment-timezone';

const TimezoneConverterComponent: React.FC = () => {
  const [time, setTime] = useState<Moment>(moment().utc());
  const [tz1, setTz1] = useState<string>('America/New_York');
  const [tz2, setTz2] = useState<string>('Australia/Sydney');

  const localTimezone = moment.tz.guess();

  const someTZOptions = [
    'Africa/Johannesburg',
    'America/New_York',
    'Asia/Shanghai',
    'Asia/Tokyo',
    'Australia/Sydney',
    'Europe/London',
    'Europe/Kiev',
  ];

  const TZSelect: React.FC<{ value: string, onChangeTimezone: (newTZ: string) => void }> = (props) => {
    return (
      <div>
        <select 
          value={props.value}
          onChange={(e) => props.onChangeTimezone(e.target.value)}
        >
          {someTZOptions.map((localeName) => {
            return <option>{localeName}</option>
          })}
        </select>
        : {moment(time).tz(props.value).format('YYYY-MM-DD h:mm:ss a z')} {/* Implicit clone to prevent mutating UTC time state */}
      </div>
    );
  };

  const handleChangeTime = (e: React.FormEvent<HTMLInputElement>) => {
    const rawTimeInput = e.currentTarget.value;
    const newHour = Number(rawTimeInput.substr(0, 2));
    const newMinute = Number(rawTimeInput.substr(3, 2));

    const newTime = moment().tz(localTimezone).hour(newHour).minute(newMinute).utc(); // time state always in UTC
    setTime(newTime);
  };

  return (
    <div>
      <p>{localTimezone}: {moment(time).tz(localTimezone).format('YYYY-MM-DD h:mm:ss a z')}</p>

      <TZSelect value={tz1} onChangeTimezone={setTz1} />

      <TZSelect value={tz2} onChangeTimezone={setTz2} />

      <div>
        Choose time (in local timezone):&nbsp;
        <input
          type='time'
          onChange={handleChangeTime}
        />
      </div>

      <div>
        <button onClick={() => setTime(moment().utc())}>
          Reset to current time
        </button>
      </div>
    </div>
  );
};
```

### Another consideration (and a big caveat): Moment.js is starting to grow grey hair

While Moment.js is a powerful library, we should keep in mind that it is also getting old. It will still be supported for the foreseeable future, although no new features will be added. In fact, the documentation for Moment.js already recommends alternative libraries for new projects. If the syntax discussed above seems reasonable to you, a solid option would be to choose Day.js instead. Day.js is essentially a lightweight version of Moment.js and optionally comes bundled with Internationalization objects for time zone support.

There are also hopes that such external libraries will become unnecessary with the potential introduction of global Temporal objects to JavaScript, which includes object types designed to work with time zones. I suppose only time will tell if working with time zones in vanilla JavaScript ends up involving fewer headaches.


#### [Link to Moment.js documentation](https://momentjs.com/docs/)
