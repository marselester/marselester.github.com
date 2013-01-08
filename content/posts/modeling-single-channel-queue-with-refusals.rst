==========================================
Моделирование одноканальной СМО с отказами
==========================================

:date: 2009-05-30
:tags: python, modeling, single-channel queue
:category: Misc
:slug: modeling-single-channel-queue-with-refusals

Дана одноканальная система массового обслуживания с отказами. В нее поступают
заявки через промежуток времени ``n``, где ``n`` – случайная величина,
подчиненная равномерному закону распределения. Время обслуживания заявки
системой ``m`` также является случайной величиной с показательным законом
распределения. Если к моменту прихода заявки канал занят, заявка покидает
систему необслуженной.

Изначально код был написан на php, вот `пример на python 3
<https://github.com/marselester/single-channel-queuing>`_:

.. code-block:: python3

    import random


    class RequestPoll:
        """Iterator that yields random requests and keeps statistic."""

        def __init__(self, time_to_finish, intensity_of_service_flow):
            self.time_to_finish = time_to_finish
            self.intensity_of_service_flow = intensity_of_service_flow

        def __str__(self):
            return (
                'Total: {}\n'.format(self.total()) +
                'Processed: {}\n'.format(self.qty_of_processed_requests) +
                'Refused: {}\n'.format(self.qty_of_refused_requests) +
                'Proportion of processed requests: {}\n'.format(self.proportion_of_processed_requests()) +
                'Probability of refuse: {}\n'.format(self.probability_of_refuse()) +
                'Absolute bandwidth: {}'.format(self.abs_bandwidth())
            )

        def __iter__(self):
            self.system_uptime = 0
            self.time_when_channel_will_be_free = 0
            self.qty_of_processed_requests = 0
            self.qty_of_refused_requests = 0
            return self

        def __next__(self):
            if self.system_uptime > self.time_to_finish:
                raise StopIteration
            time_when_request_came_in = random.random()
            self.system_uptime += time_when_request_came_in

            if self._can_system_process_request():
                self.qty_of_processed_requests += 1

                time_for_which_request_has_taken_channel = random \
                    .expovariate(self.intensity_of_service_flow)
                self.time_when_channel_will_be_free = self.system_uptime \
                    + time_for_which_request_has_taken_channel

                return 'request added to queue at {}'.format(self.system_uptime)
            else:
                self.qty_of_refused_requests += 1

                return 'requests refused at {}'.format(self.system_uptime)

        def _can_system_process_request(self):
            return self.system_uptime >= self.time_when_channel_will_be_free

        def total(self):
            return self.qty_of_processed_requests + self.qty_of_refused_requests

        def abs_bandwidth(self):
            return self.qty_of_processed_requests / self.time_to_finish

        def proportion_of_processed_requests(self):
            return self.qty_of_processed_requests / self.total()

        def probability_of_refuse(self):
            return self.qty_of_refused_requests / self.total()

    intensity_of_service_flow = 0.5
    time_to_finish = 70
    request_poll = RequestPoll(time_to_finish, intensity_of_service_flow)

    for request in request_poll:
        print(request)

    print(request_poll)
