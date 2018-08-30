    /*
     * N.B. Why we have this evil global class
     *
     * In a WPF or UWP application, most commands must have the Dispatcher
     * scheduler set, because notifications will end up being run on another thread;
     * this happens most often in a CanExecute observable. Unfortunately, in a Unit
     * Test framework, while the MS Test Unit runner will *set* the Dispatcher (so
     * we can't even use the lack of its presence to determine whether we're in a
     * test runner or not), none of the items queued to it will ever be executed
     * during the unit test.
     *
     * Initially, I tried to plumb the ability to set the scheduler throughout the
     * classes, but when you start building applications on top of that, having to
     * have *every single* class have a default Scheduler property is really
     * irritating, with either default making life difficult.
     *
     * This class also initializes a whole bunch of other stuff, including the IoC container,
     * logging and error handling.
     */
    public static class RxApp