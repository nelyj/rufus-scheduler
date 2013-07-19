
# rufus-scheduler

[![Build Status](https://secure.travis-ci.org/jmettraux/rufus-scheduler.png)](http://travis-ci.org/jmettraux/rufus-scheduler)

Job scheduler for Ruby (at, cron, in and every jobs).

**Warning**: this is the README of the 3.0 line of rufus-scheduler. It got promoted to master branch on 2013/07/15. Head to the [2.0 line's README](https://github.com/jmettraux/rufus-scheduler/blob/two/README.rdoc) if necessary (if your rufus-scheduler version is 2.0.x).

(When the 3.0 gem is released, this warning will get removed).

Quickstart:
```
require 'rufus-scheduler'

scheduler = Rufus::Scheduler.new

scheduler.in '3s' do
  puts 'Hello... Rufus'
end

scheduler.join
  # let the current thread join the scheduler thread
```

Various forms of scheduling are supported:
```
require 'rufus-scheduler'

scheduler = Rufus::Scheduler.new

# ...

scheduler.in '10d' do
  # do something in 10 days
end

scheduler.at '2030/12/12 23:30:00' do
  # do something at a given point in time
end

scheduler.every '3h' do
  # do something every 3 hours
end

scheduler.cron '5 0 * * *' do
  # do something every day, five minutes after midnight
  # (see "man 5 crontab" in your terminal)
end

# ...
```


## note about the 3.0 line

It's a complete rewrite of rufus-scheduler.

There is no EventMachine-based scheduler anymore.


## Notables changes:

* As said, no more EventMachine-based scheduler
* ```scheduler.every('100') {``` will schedule every 100 seconds (previously, it would have been 0.1s). This aligns rufus-scheduler on Ruby's ```sleep(100)```
* The scheduler isn't catching the whole of Exception anymore, only StandardException
* Rufus::Scheduler::TimeOutError renamed to Rufus::Scheduler::TimeoutError


## scheduling

TODO: in/at/cron/every


## pause and resume the scheduler

The scheduler can be paused via the #pause and #resume methods. One can determine if the scheduler is currently paused by calling #paused?.

While paused, the scheduler still accepts schedules, but no schedule will get triggered as long as #resume isn't called.

TODO: :discard_the_past => true?


## job options

### :blocking => true

By default, jobs are triggered in their own, new thread. When :blocking => true, the job is triggered in the scheduler thread (a new thread is not created). Yes, while the job triggers, the scheduler is not scheduling.

### :overlap => false

Since, by default, jobs are triggered in their own new thread, job instances might overlap. For example, a job that takes 10 minutes and is scheduled every 7 minutes will have overlaps.

To prevent overlap, one can set :overlap => false. Such a job will not trigger if one of its instance is already running.

### :mutex => mutex_instance / mutex_name / array of mutexes

When a job with a mutex triggers, the job's block is executed with the mutex around it, preventing other jobs with the same mutex to enter (it makes the other jobs wait until it exits the mutex).

This is different from :overlap => false, which is, first, limited to instances of the same job, and, second, doesn't make the incoming job instance block/wait but give up.

:mutex accepts a mutex instance or a mutex name (String). It also accept an array of mutex names / mutex instances. It allows for complex relations between jobs.

Array of mutexes: original idea and implementation by [Rainux Luo](https://github.com/rainux)

### :timeout => duration or point in time

It's OK to specify a timeout when scheduling some work. After the time specified, it gets interrupted via a Rufus::Scheduler::TimeoutError.

```ruby
  scheduler.in '10d', :timeout => '1d' do
    begin
      # ... do something
    rescue Rufus::Scheduler::TimeoutError
      # ... that something got interrupted after 1 day
    end
  end
```

The :timeout option accepts either a duration (like "1d" or "2w3d") or a point in time (like "2013/12/12 12:00").


## Job methods

When calling a schedule method, the id (String) of the job is returned. Longer schedule methods return Job instances directly. Calling the shorter schedule methods with the :job => true also return Job instances instead of Job ids (Strings).

```ruby
  require 'rufus-scheduler'

  scheduler = Rufus::Scheduler.new

  job_id =
    scheduler.in '10d' do
      # ...
    end

  job =
    scheduler.schedule_in '1w' do
      # ...
    end

  job =
    scheduler.in '1w', :job => true do
      # ...
    end
```

Those Job instances have a few interesting methods / properties:

### id, job_id

Returns the job id.

```ruby
job = scheduler.schedule_in('10d') do; end
job.id
  # => "in_1374072446.8923042_0.0_0"
```

### opts

Returns the options passed at the Job creation.

```ruby
job = scheduler.schedule_in('10d', :tag => 'hello') do; end
job.opts
  # => { :tag => 'hello' }
```

### original

Returns the original schedule.

```ruby
job = scheduler.schedule_in('10d', :tag => 'hello') do; end
job.original
  # => '10d'
```

### scheduled_at

Returns the Time instance when the job got created.

```ruby
job = scheduler.schedule_in('10d', :tag => 'hello') do; end
job.scheduled_at
  # => 2013-07-17 23:48:54 +0900
```

### last_time

Returns the last time the job triggered (is usually nil for AtJob and InJob).
k
```ruby
job = scheduler.schedule_every('1d') do; end
# ...
job.scheduled_at
  # => 2013-07-17 23:48:54 +0900
```

### unschedule

Unschedule the job, preventing it from firing again and removing it from the schedule. This doesn't prevent a running thread for this job to run until its end.

### threads, thread_values

Returns the list of threads currently "hosting" runs of this Job instance.

Thread values returns the info stored under the job key in their thread local variables.

### kill

Kills all the currently running threads hosting runs of this Job instance.

Nota bene: this doesn't unschedule the Job instance.

### running?

Returns true if there is at least one running Thread hosting a run of this Job instance.

### tags

Returns the list of tags attached to this Job instance.

By default, returns an empty array.

```ruby
job = scheduler.schedule_in('10d') do; end
job.tags
  # => []

job = scheduler.schedule_in('10d', :tag => 'hello') do; end
job.tags
  # => [ 'hello' ]
```

## AtJob and InJob methods
### time

## EveryJob methods
### frequency
### next_time

## CronJob methods


## looking up jobs

### Scheduler#job(job_id)

The scheduler #job(job_id) method can be used to lookup Job instances.

```ruby
  require 'rufus-scheduler'

  scheduler = Rufus::Scheduler.new

  job_id =
    scheduler.in '10d' do
      # ...
    end

  # later on...

  job = scheduler.job(job_id)
```

### Scheduler #jobs #at_jobs #in_jobs #every_jobs and #cron_jobs

Are methods for looking up lists of scheduled Job instances.

Here is an example:

```ruby
  #
  # let's unschedule all the at jobs

  scheduler.at_jobs.each(&:unschedule)
```

### Scheduler#jobs(:tag / :tags => x)

When scheduling a job, one can specify one or more tags attached to the job. These can be used to lookup the job later on.

```ruby
  scheduler.in '10d', :tag => 'main_process' do
    # ...
  end
  scheduler.in '10d', :tags => [ 'main_process', 'side_dish' ] do
    # ...
  end

  # ...

  jobs = scheduler.jobs(:tag => 'main_process')
    # find all the jobs with the 'main_process' tag

  jobs = scheduler.jobs(:tags => [ 'main_process', 'side_dish' ]
    # find all the jobs with the 'main_process' AND 'side_dish' tags
```

### Scheduler#running_jobs

Returns the list of Job instance that have currently running instances.

Whereas other "_jobs" method scan the scheduled job list, this method scans the thread list to find the job. It thus comprises jobs that are running but are not scheduled anymore (that happens for at and in jobs).


## misc Scheduler methods

### Scheduler#unschedule(job_or_job_id)

Unschedule a job given directly or by its id.

### Scheduler#terminate_all_jobs

Unschedules all the jobs, then block until all the jobs that were running terminate.

### Scheduler#shutdown

Shuts down the scheduler, ceases any scheduler/triggering activity.

### Scheduler#shutdown(:terminate)

Calls Scheduler#terminate_all_jobs then shuts down the scheduler. That means this shutdown variant blocks until all the jobs are terminated and then shuts down.

### Scheduler#shutdown(:kill)

Kills all the job (threads) and then shuts the scheduler down. Radical.

### Scheduler#join

Let's the current thread join the scheduling thread in rufus-scheduler. The thread comes back when the scheduler gets shut down.


## Rufus::Scheduler.new options

### :frequency

By default, rufus-scheduler sleeps 0.300 second between every step. At each step it checks for jobs to trigger and so on.

The :frequency option lets you change that 0.300 second to something else.

```ruby
  scheduler = Rufus::Scheduler.new(:frequency => 5)
```

It's OK to use a time string to specify the frequency.

```ruby
  scheduler = Rufus::Scheduler.new(:frequency => '2h10m')
    # this scheduler will sleep 2 hours and 10 minutes between every "step"
```

Use with care.


## parsing cronlines and time strings

TODO


## license

MIT, see LICENSE.txt
