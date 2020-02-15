#!/usr/bin/python

import abc
import datetime
import math
import signal
import time
import threading


class MyException(Exception, metaclass=abc.ABCMeta):
    message = ''

    def __init__(self, **kwargs):
        self.msg = self.message % kwargs
        super(MyException, self).__init__(self.msg)


class TimerTimeout(MyException):
    message = 'Timer timeout expired after %(timeout)s second(s).'


class SingletonDecorator:

    def __init__(self, klass):
        self.klass = klass
        self.instance = None

    def __call__(self, *args, **kwargs):
        if self.instance is None:
            self.instance = self.klass(*args, **kwargs)
        return self.instance


class Alarm(object):
    """Alarm object"""

    def __init__(self, set_time, timeout, _id, signal_handler):
        self.set_time = set_time
        self.timeout = timeout
        self.id = _id  # ID of the Time class instantiating this Alarm
        self.signal_handler = signal_handler

    def sooner(self, other_alarm, current_time=None):
        """Compares self with other alarm

        Returns True if current alarm is going tu buzz before other_alarm,
        based on the set_time and the timeout.
        """
        current_time = current_time or other_alarm.set_time
        return (self.actual_timeout(current_time) <
                other_alarm.actual_timeout(current_time))

    def actual_timeout(self, current_time=None):
        """Remaining timeout based on the set_time and the timeout"""
        current_time = current_time or time.time()
        return self.timeout - (current_time - self.set_time)


@SingletonDecorator
class _AlarmClock(object):
    """Singleton Alarm Clock"""

    def __init__(self):
        self._running_alarm = []  # *Ordered* running alarm list.
        self._lock = threading.Lock()
        self._is_running = False
        self._old_handler = None

    @property
    def remaining_time(self):
        return signal.getitimer(signal.ITIMER_REAL)[0]

    def _set_old_handler(self):
        if not self._old_handler:
            self._old_handler = signal.getsignal(signal.SIGALRM)

    def _insert_alarm(self, new_alarm):
        """Insert a new Alarm in _running_alarm, ordered by actual_timeout

        Returns the position of the the Alarm in the ordered list of running
        alarms.
        """
        idx = -1
        for idx, alarm in enumerate(self._running_alarm):
            if new_alarm.sooner(alarm):
                break
        else:
            idx += 1
        self._running_alarm.insert(idx, new_alarm)
        return idx

    def _pop_alarm(self, caller_id):
        """Removes an Alarm from the _running_alarm list, using the id"""
        # This method should be called from inside a lock context.
        for idx, alarm in enumerate(self._running_alarm):
            if alarm.id == caller_id:
                break
        else:
            return
        return self._running_alarm.pop(idx)

    def _set_next_alarm(self, caller_id=None):
        """Stop an Alarm (buzzing or not) and set the next one or reset"""
        with self._lock:
            alarm_buzzing = None
            if self._running_alarm:
                if caller_id:
                    alarm_buzzing = self._pop_alarm(caller_id)
                else:
                    alarm_buzzing = self._running_alarm.pop(0)

            if self._running_alarm:
                self._set_alarm_signal(self._running_alarm[0])
            else:
                self._reset_alarm_signal()
        return alarm_buzzing

    def _timeout_handler(self, *args, **kwargs):
        """Timeout handler"""
        alarm_buzzing = self._set_next_alarm()
        # The Alarm signal handler call must be done outside the lock context
        alarm_buzzing.signal_handler()

    def _set_alarm_signal(self, alarm):
        """Set an active Alarm"""
        self._set_old_handler()
        signal.signal(signal.SIGALRM, self._timeout_handler)
        # NOTE: the precision is at integer level. It is not possible to set
        # a fraction of second.
        signal.alarm(math.ceil(alarm.actual_timeout()))
        self._is_running = True

    def _reset_alarm_signal(self):
        """Stop the Clock"""
        signal.alarm(0)
        self._is_running = False

    def set_alarm(self, timeout, caller_id, signal_handler):
        """Set a new Alarm in the Clock

        If there is another Alarm running and it's going to buzz before the
        new one
        """
        with self._lock:
            new_alarm = Alarm(time.time(), timeout, caller_id, signal_handler)
            idx = self._insert_alarm(new_alarm)
            if idx == 0:
                self._set_alarm_signal(new_alarm)

    def stop_alarm(self, caller_id):
        """Stops a running Alarm

        When the Timer context finishes without reaching the timeout, it can
        gracefully stop the alarm. If the list of ordered running alarms is not
        empty, the next one will be set.
        """
        self._set_next_alarm(caller_id)


class Timer(object):
    """Timer context manager class

    Derived from Timer class in neutron.common.utils:
    https://github.com/openstack/neutron/blob/
      a026b39617f4c2d29f2903909fb65f7adb327fa7/neutron/common/utils.py#L866

    This class creates a context that:
    - Triggers a timeout exception if the timeout is set.
    - Returns the time elapsed since the context was initialized.
    - Returns the time spent in the context once it's closed.

    The timeout exception can be suppressed; when the time expires, the context
    finishes without rising TimerTimeout.
    """
    def __init__(self, timeout=None, raise_exception=True):
        self.start = self.delta = None
        self._timeout = int(timeout) if timeout else None
        self._timeout_flag = False
        self._raise_exception = raise_exception
        self._alarm = _AlarmClock()

    def _timeout_handler(self, *args, **kwargs):
        self._timeout_flag = True
        if self._raise_exception:
            raise TimerTimeout(timeout=self._timeout)

    def __enter__(self):
        self.start = datetime.datetime.now()
        if self._timeout:
            self._alarm.set_alarm(self._timeout, id(self),
                                  self._timeout_handler)
        return self

    def __exit__(self, exc, value, traceback):
        self.delta = datetime.datetime.now() - self.start
        if self._timeout:
            self._alarm.stop_alarm(id(self))

    def __getattr__(self, item):
        return getattr(self.delta, item)

    def __iter__(self):
        self._raise_exception = False
        return self.__enter__()

    def __next__(self):
        return self.next()

    @property
    def delta_time_sec(self):
        return (datetime.datetime.now() - self.start).total_seconds()


###############################################################################
# TESTS #
###############################################################################


def print_time():
    # If the remainin time is 0.0, then the alarm clock is stopped.
    print('  --> Remaining time in running alarm: %s' %
          _AlarmClock().remaining_time)


def wait_time():
    print_time()
    time.sleep(1)


print('\n\nExample #1. Timer with exception, context exists before timeout.')
with Timer(timeout=20) as timer:
    wait_time()
    wait_time()

print('Time used inside context: %s' % timer.delta)
wait_time()


print('\n\nExample #2. Timer with exception, context does not exist in time.')
try:
    with Timer(timeout=2) as timer:
        wait_time()
        wait_time()
        wait_time()
except TimerTimeout:
    print('Timeout 2 seconds')
    print('Time used inside context: %s' % timer.delta)


print('\n\nExample #3. Timer without exception, context does not exist in '
      'time.')
with Timer(timeout=2, raise_exception=False) as timer:
    wait_time()
    wait_time()
    wait_time()

print('Time used inside context: %s' % timer.delta)


print('\n\nExample #4. Outer Timer without exception, inner Timer with '
      'exception.')
with Timer(timeout=20, raise_exception=False) as timer1:
    print('Outer timer')
    wait_time()
    try:
        with Timer(timeout=2) as timer2:
            print('Inner timer')
            wait_time()
            wait_time()
            wait_time()
    except TimerTimeout:
        print('Inner timer exception')
        print('Timeout 2 seconds')
        print('Time used inside context: %s' % timer2.delta)
        wait_time()
        wait_time()

    print('Outer timer again')
    wait_time()
    wait_time()

print('Time used inside outer context: %s' % timer1.delta)
