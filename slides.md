!SLIDE[bg=images/surfheadstand.jpg]
### Jacob Burkhart
<br/><br/><br/><br/>
<br/><br/><br/><br/>
<br/><br/><br/><br/><br/>
## [@beanstalksurf](http://twitter.com/beanstalksurf)

.notes change to to beanstalksurf.jpg when in Japan

!SLIDE[bg=images/closeout.jpg] black
### How to Fail at Background Jobs
## `jacobo.github.com/background_jobs`

.notes Failure is a good thing. We more from failure than success.

!SLIDE[bg=images/evan.jpg] moredarkness shadowh2
### Delayed::Job

<br/><br/><br/><br/><br/><br/><br/>
<br/><br/><br/><br/><br/><br/>

## try this instead: (requires postgresql) `github.com/ryandotsmith/queue_classic`

!SLIDE[bg=images/distill.jpg]
### The End
<br/><br/><br/><br/>
<br/><br/><br/><br/><br/>
<br/><br/><br/><br/><br/>
## `distill.engineyard.com`
## CFP closes Tuesday, April 9

!SLIDE[bg=images/closeout.jpg] black
### Seeking Better Abstractions for Background Jobs
<br/><br/><br/><br/><br/><br/><br/><br/>
## `jacobo.github.com/background_jobs`

!SLIDE[bg=images/skimboard.jpg] smallerh2 moredarkness
### Rails 4 Queuing
<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>
## `github.com/rails/rails/commit/f9da785d0b1b22317cfca25c15fb555e9016accb`

!SLIDE
### The API

    @@@ ruby
    class MyJob

      def initialize(account)
        @account = account
      end

      def run
        puts "working on #{@account.name}..."
      end

    end

    Rails.queue[:jobs].push(MyJob.new(account))

.notes Rails core says this is a failure because the Q name is in the caller instead of being defined in the job
.notes I say this is a fail because this API places terrible constraints and limitations on backend integrators

!SLIDE[bg=images/marshal.png]
### Fail at Serialization
&nbsp;

!SLIDE
### Solve the wrong problem

    @@@ ruby
    class BodyProxy
      def each
        @body.each{|x| yield x}
        while(email = CleverMailer.emails_to_send.pop)
          email.deliver
        end
      end
    end

    def call(env)
      status, headers, body = @app.call(env)
      headers["Content-Length"] = body.map(&:bytesize).sum
      [status, headers, BodyProxy.new(body)]
    end

!SLIDE
### Will be <s><div></div>revisited</s> <br/> re-written in 4.1

## `github.com/rails/rails/pull/9910`
## `github.com/rails/rails/pull/9924`

!SLIDE[bg=images/skimboardfail.jpg]
### Moving on...

!SLIDE
### 2009

![](images/3m-lava-chairside-oral-scanner.jpg)

!SLIDE
### Starling, Workling

    @@@ ruby
    class FileCopier < Workling::Base
      def copy_case_files(options)
        ...
      end
    end
    FileCopier.async_copy_case_files(...)

    Workling::Remote.dispatcher = 
      Workling::Remote::Runners::StarlingRunner.new

&nbsp;

    @@@ ruby
    starling -f config/starling.yml
    script/workling_client run

!SLIDE[bg=images/failwhale.png]
&nbsp;

.notes Stolen from: http://www.subcide.com/experiments/fail-whale/

!SLIDE[bg=images/rabbitmq.png]
&nbsp;

.notes http://www.rabbitmq.com/
.notes "Robust" only when you setup your queues and topics and exchanges correctly, and set them to be durable, and send durable messages, and send acks.
.notes so we made sore rabbit was sending and handling messages reliably because we were told this is a feature of rabbit. not because it's something we thought we needed.  In my experience, generally you have many more problems with message execution, than you do with message delivery.

!SLIDE
### AMQP

    @@@ ruby
    connection = AMQP.connect(:host => '127.0.0.1')

    channel  = AMQP::Channel.new(connection)
    queue    = channel.queue("some.q")
    exchange = channel.default_exchange

    queue.subscribe do |payload|
      puts "Received a message: #{payload}"
    end

    exchange.publish "Hello, world!", 
                     :routing_key => queue.name

## `github.com/ruby-amqp/amqp`
## `github.com/ruby-amqp/bunny`

!SLIDE[bg=images/worklingrunners.png]
&nbsp;

.notes Brontes fork of workling at: https://github.com/brontes3d/workling

!SLIDE[bg=images/bug.jpg]
### Little Bug

!SLIDE
### ActiveRecord:: RecordNotFound

    @@@ ruby
    class Device < ActiveRecord::Base
      after_create do |d|
        BackgroundJob.async_run(d.id)
      end
    end

    class BackgroundJob
      def run(device_id)
        #sleep 1 ?
        Device.find(device_id)
      end
    end


!SLIDE
### Hack ActiveRecord

    @@@ ruby
    class Device < ActiveRecord::Base
      after_create do |d|
        d.commit_callback do
          BackgroundJob.async_run(d.id)
        end
      end
    end

    ...
    def commit_callback
      self.connection.instance_eval do
        class << self
          alias commit_db_transaction_original commit_db_transaction
          ...

## `github.com/brontes3d/commit_callback`
(rails 2.3 only)

.notes probably only works with rails 2.3 etc...

!SLIDE[bg=images/manybugs.jpg] moredarkness bullets incremental bigger-bullets
### More Bugs
<br/><br/><br/>

* Fundamental Flaw

.notes generic abstractions are hard
.notes poll vs. push
.notes because workling controls the run loop, we couldn't easily mix with EM-based xmpp
.notes spinning up (and down) an event machine every time you need to send a message is really crappy

!SLIDE
### Do it Yourself

    @@@ ruby
    class FileCopier < AmqpListener::Listener
      subscribes_to :case_file_copy_requests

      def handle(options)
        ...
      end
    end

    FileCopier.notify(...)

&nbsp;

    @@@ ruby
    script/amqp_listener run

## `github.com/brontes3d/amqp_listener`

!SLIDE[bg=images/sunset.jpg] align-left
### Moment of Reflection

.notes Did the tools fail us?
.notes Did we fail at using them?
.notes stray from best practices leads to re-writing things from scratch. leads to being on an island

!SLIDE[bg=/images/engineyardcloud.png]
### Trains
### ... and Resque

!SLIDE
### Boot an EC2 Server

    @@@ ruby
    class InstanceProvision

      def self.perform(instance_id)
        instance = Instance.find(instance_id)

        fog = Fog::Compute.new(...)
        server = fog.servers.create(...)
        instance.amazon_id = server.id

        while(!server.ready?)
          sleep 1
          server.reload
        end

        instance.attach_ip!
        ...

!SLIDE
### Extractable?

    @@@ ruby
    class InstanceProvision

      def self.perform(aws_creds, create_params, callback_url)
        fog = Fog::Compute.new(aws_creds)
        server = fog.servers.create(create_params)
        API.post(callback_url, :instance_amazon_id => server.id)

        while(!server.ready?)
          sleep 1
          server.reload
        end

        ip = fog.ips.create!
        ip.server = server
        API.post(callback_url, :attached_ip_amazon_id => ip.id)
        ...

!SLIDE
### Generalizable?

    @@@ ruby
    class MethodCalling
      def self.perform(class_str, method, id, *args)
        model_class = Object.const_get(class_str)
        model = model_class.find(id)
        model.send(method, *args)
      end
    end

    class Instance
      def provision
        Resque.enqueue(MethodCalling, Instance, :provision!, id)
      end

      def provision!
        #actually do it
      end
    end

!SLIDE
### Async

    @@@ ruby
    require 'async'
    require 'async/resque'
    Async.backend = Async::ResqueBackend

    class Instance < ActiveRecord::Base
      def provision(*args)
        Async.run{ provision_now(*args)}
      end
      def provision_now(*args)
        #actually do it
      end
    end

## `github.com/engineyard/async`

!SLIDE[bg=images/instancehang.png]
&nbsp;

.notes We have customers complaining

!SLIDE[bg=images/resquehang.png]
&nbsp;

.notes We would have jobs fail, and have to go in and read the code of this method body, and figure out where it failed... and try to fix it. But is the job still running? Did the job throw an exception?

!SLIDE bullets incremental
### Reliability
* Reliable Queue vs Reliable Job
* Retry (raise, crash, hang). Time-out
* Monitor / Maintain N workers
* Graceful Restart
* Did it run? Did it fail? How? Where?

!SLIDE
### Make a Resque Plugin

    @@@ ruby
    class InstanceProvision
      extend Resque::Plugins::JobTracking

      def self.track(instance_id, opts)
        i = Instance.find(instance_id)
        ["Account:#{i.account_id}",
         "Instance:#{instance_id}"]
      end

      def self.perform(instance_id, opts)
        #do stuff
      end
    end

## `github.com/engineyard/resque-job-tracking`

!SLIDE
### Helps a little?

    @@@ ruby
    InstanceProvision.pending_jobs("Instance:532")

    InstanceProvision.failed_jobs("Account:121")

!SLIDE[bg=images/job_dependencies.png]
### Jobs Dependencies

!SLIDE
### Make another

    @@@ ruby
    class Sandwich
      extend Resque::Plugins::Delegation

      def self.steps(tomato_color, cheese_please, cheesemaker)
        step "fetch a", :tomato do
          depend_on(Tomato, tomato_color)
        end
        step "slice the ", :tomato, " and make", :tomato_slices do |tomato|
          tomato.split(",")
        end
        step "fetch the", :cheese_slices do
          if cheese_please
            depend_on(Cheese, cheesemaker)
            ...

## `github.com/engineyard/resque-delegation`

!SLIDE
### Desperation?

    @@@ ruby
    class ResqueJob < ActiveRecord::Base
    end

&nbsp;

    @@@ ruby
    class AbstractJob

      def self.store_meta(meta)
        meta_id = meta["meta_id"]
        ResqueJob.create!(meta.slice(
          "meta_id", "job_class",
          "enqueued_at", "started_at", "finished_at"))
        super(meta)
      end

    ...

!SLIDE
### Maybe we just need better logging?

<h2><iframe width="640" height="360" src="http://www.youtube.com/embed/NpTT30wLL-w?rel=0" frameborder="0" allowfullscreen></iframe></h2>

## [`http://youtu.be/NpTT30wLL-w`](http://youtu.be/NpTT30wLL-w)

!SLIDE[bg=images/josh.jpg] moredarkness
### Model Intent

!SLIDE
### Data belongs in a database (not redis)

    @@@ ruby
    class InstanceProvision < ActiveRecord::Base
      belongs_to :instance

      def run(instance_id)
        fog = Fog::Compute.new(...)
        server = fog.servers.create(...)
        instance.amazon_id = server.id

        while(!server.ready?)
          sleep 1
          server.reload
        end

        instance.attach_ip!
        ...

!SLIDE
# `requested_at`
# `started_at`
# `finished_at`
# `state`

!SLIDE
### Idempotent

    @@@ ruby
    class InstanceProvision < ActiveRecord::Base
      belongs_to :instance

      def run(instance_id)
        with_lock do
          if self.state == "running"
            raise
          else
            self.state = "running"
            self.started_at = Time.now
            save!
          end
        end
        ...

!SLIDE
### Non-Resque specific job-middleware

    @@@ ruby
    procedure = Viaduct::Builder.new do
      use Instrumetation
      use ProvisionNotification
      use ProvisionInstance
    end
    Viaduct::Runner.new.run(procedure, data)

## `github.com/slack/viaduct`

!SLIDE[bg=images/sunset3.jpg] align-left
### Moment of Reflection

.notes at 3M we fixed our problems by making the Queuing system smarter.
.notes I tried to do this at EY and failed, so we fixed out problems by using a dumber Q and smarter business logic

!SLIDE[bg=images/peaches.jpg] moredarkness bullets incremental bigger-bullets
### 3 Ingredients
* Work Loop
* Monitor & Restart
* Queue

!SLIDE[bg=images/maze.jpg]
### Loop

!SLIDE
### Resque Event Loop

    @@@ ruby
    def work(interval = 5, &block)
      loop do
        run_hook :before_fork, job

        if @child = fork
          procline "Forked #{@child} at #{Time.now.to_i}"
          Process.wait
        else
          procline "Processing #{job.queue} since #{Time.now.to_i}"
          perform(job, &block)
          exit! unless @cant_fork
        end
      end

`github.com/defunkt/resque/blob/master/lib/resque/worker.rb`

!SLIDE
### EventMachine

    @@@ C
    void EventMachine_t::Run()
      //Epoll and Kqueue stuff..
      ...

      while (true) {
        _UpdateTime();
        _RunTimers();

        _AddNewDescriptors();
        _ModifyDescriptors();

        _RunOnce();
        if (bTerminateSignalReceived)
          break;
      }
    }

`github.com/eventmachine/eventmachine/blob/master/ext/em.cpp`


!SLIDE
### EM.next_tick

    @@@ ruby
    require 'eventmachine'
    EM.run {
      EM.start_server(host, port, self)
    }

    EM.next_tick{ puts "do something" }

!SLIDE
### Celluloid / Sidekiq

    @@@ ruby
    class Sidekiq::Manager
      include Celluloid
      ???

    class Sidekiq::Fetcher
      include Celluloid
      ???

    class Sidekiq::Processor
      include Celluloid
      ???

# [`sidekiq.org`](http://sidekiq.org)
## `github.com/celluloid/celluloid`

!SLIDE[bg=images/riding.jpg]
### Monitor & Restart

.notes graceful restart
.notes kill old dead things

!SLIDE align-left
### Resque Signals

## `QUIT` - Wait for child to finish then exit

## `TERM` / `INT` - Immediately kill child, exit

## `USR1` - Immediately kill child, don't exit

## `USR2` -Don't start to process new jobs

## `CONT` - Start to process new jobs again after a USR2

!SLIDE
### Monitor with God

    @@@ ruby
    5.times do |n|
      God.watch do |w|
        w.name     = "resque-#{num}"
        w.group    = 'resque'
        w.interval = 30.seconds
        w.log      = "#{app_root}/log/worker.#{num}.log"
        w.dir      = app_root
        w.env      = {
          "GOD_WATCH"   => w.name,
          "QUEUE"       => '*'
        }
        w.start    = "bundle exec rake --trace resque:work"
      ...

# [`godrb.com`](http://godrb.com)

!SLIDE
### Or Daemons

    @@@ ruby
    require 'daemons'

    options = {
      :app_name => "worker",
      :log_output => true,
      :backtrace => true,
      :dir_mode => :normal,
      :dir => File.expand_path('../../tmp/pids',  __FILE__),
      :log_dir => File.expand_path('../../log',  __FILE__),
      :multiple => true,
      :monitor => true
    }

    Daemons.run(File.expand_path('../worker',  __FILE__), options)

# [`daemons.rubyforge.org`](http://daemons.rubyforge.org)

!SLIDE
### Or Don't?

![](images/torquebox.jpg)

# [`torquebox.org`](http://torquebox.org)

!SLIDE[bg=images/chris.jpg]
&nbsp;

!SLIDE[bg=images/lineup.jpg]
### Queue

!SLIDE
### Thread-safe Queue
    @@@ ruby
    module BundlerApi
      class ConsumerPool

        def initialize
          @queue = Queue.new
        end

        def enq(job)
          @queue.enq(job)
        end

        ...
          job = @queue.deq
          job.run

`github.com/rubygems/bundler-api/blob/master/lib/ bundler_api/update/consumer_pool.rb`


!SLIDE
### DRB objects

    @@@ ruby
    class MultiHeadedGreekMonster
      class ServiceManager
        def initialize(progress)
          @things = []
        end
        def give(thing)
          @things.unshift(thing)
        end
        def take
          @things && @things.pop
        end

    ...
    DRb.start_service "druby://localhost:#{@on_port}", 
                      ServiceManager.new(@progress)

`github.com/engineyard/multi_headed_greek_monster/blob/master/ lib/multi_headed_greek_monster.rb`

!SLIDE
### Qu Push/Pop Redis

    @@@ ruby
    payload.id = SimpleUUID::UUID.new.to_guid
    redis.set("job:#{payload.id}", 
      encode('klass' => payload.klass.to_s, 
             'args' => payload.args))
    redis.rpush("queue:#{payload.queue}", payload.id)
    redis.sadd('queues', payload.queue)

    ...

    redis.lpop(queue)

`github.com/bkeepers/qu/blob/master/lib/qu/backend/redis.rb`

!SLIDE
### Qu Push/Pop Mongo

    @@@ ruby
    payload.id = SimpleUUID::UUID.new.to_guid
    payload.id = BSON::ObjectId.new
    jobs(payload.queue).insert({
      '_id' => payload.id, 
      'klass' => payload.klass.to_s, 'args' => payload.args})
    self[:queues].update({'name' => payload.queue}, 
      {'name' => payload.queue}, 'upsert' => true)

    ...

    if doc = jobs(queue).find_and_modify(:remove => true)
      doc['id'] = doc.delete('_id')
      return Payload.new(doc)
    end

`github.com/bkeepers/qu/blob/master/lib/qu/backend/mongo.rb`


!SLIDE[bg=images/handstands.jpg]
### All Together

!SLIDE

## `script/invoice_worker`

    @@@ ruby
    #!/usr/bin/env ruby
    require File.expand_path('../../config/environment',
      __FILE__)

    TrapLoop.start do
      InvoiceProcessingTask.process_invoices!
    end

## `script/invoice_runner start`
## `script/invoice_runner stop`

    @@@ ruby
    require 'daemons'
    Daemons.run(File.expand_path('../invoice_worker',  __FILE__), ...)

!SLIDE
    @@@ ruby
    class TrapLoop
      trap('TERM') { stop! }
      trap('INT')  { stop! }
      trap('SIGTERM') { stop! }

      def self.stop!
        @loop = false
      end

      def self.safe_exit_point!
        if @started && !@loop
          raise Interrupt
        end
      end

      def self.start(&block)
        @started = true
        @loop = true
        while(@loop) do
          yield
        end
      end


!SLIDE

    @@@ ruby
    class InvoiceProcessingTask
      def self.enq_task(task_id, invoice_id)
        REDIS.rpush("tasks:#{invoice_id}", task_id)
        REDIS.rpush("invoices", invoice_id)
      end

      def self.process_invoices!
        while(invoice_id = REDIS.lpop("invoices"))
          process_tasks!(invoice_id)
          TrapLoop.safe_exit_point!
        end
      end

      def self.process_tasks!(invoice_id)
        while(task_id = REDIS.lpop("tasks:#{invoice_id}"))
          if task = InvoiceProcessingTask.find_by_id(task_id)
            task.process!
            TrapLoop.safe_exit_point!
          end
        end
      end

!SLIDE[bg=images/sunset2.jpg] moredarkness incremental bullets biggish-bullets bigger-bullets
### Conclusions

  * Abstraction Awareness
  * Model important jobs in your domain
  * Contribute to Resque

!SLIDE[bg=images/mudbath.jpg] shadowh2
### Questions?
<br/><br/><br/><br/><br/>
<br/><br/><br/><br/><br/>
## `jacobo.github.com/background_jobs`
## [@beanstalksurf](http://twitter.com/beanstalksurf)
## `jacobo.github.com/talks`