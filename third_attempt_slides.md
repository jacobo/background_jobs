
!SLIDE
# Failure is a good thing


!SLIDE
# Rails 4 Q API


!SLIDE
# Fail at Abstraction

.notes We're always iterating. We're always saying "this old code sucks", let's make it better. And the more time we spend fixing it the more we discover about all of ways in which it sucks.

!SLIDE
# Resque sucks

1. Try to decouple ourselves as much as possible from Resque so that when we find a better solution, it's easy to switch.

2. Try to make Resque better. (Which means building more abstractions into Resque and thus coupling ourselves to it more)

We coupled ourselves to resque:
https://github.com/engineyard/resque-unique-job

    @@@ ruby
    class SSOCaching
      extend Resque::Plugins::UniqueJob
      @unique = true

      def self.perform(key)
        EY::SSO::User.get(key).reload!
      end
    end

    Resque.enqueue SSOCaching, 123

It might have been better to couple ourselves to the "eventually-will-exist" Rails Q API

Or use some other "abstraction on to top Resque", or write our own.

Or we could have written the abstraction ourselves:

    @@@ ruby
    class SSOCaching
      def self.enqueue(key)
        if UniqueJob.aquire_lock("SSOCaching#{key}")
          Resque.enqueue SSOCaching, key
        end
      end

      def self.perform(key)
        UniqueJob.release_lock!("SSOCaching#{key}")
        EY::SSO::User.get(key).reload!
      end
    end

    SSOCaching.enqueue 123


... and then our test framework at the time wasn't running the Resque hook chain


https://github.com/jacobo/async


!SLIDE
# Fail at Reliability

