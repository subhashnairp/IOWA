# Porting IOWA

IOWA has been designed to be easily ported to a new platform. All functions contained in the [`System Abstraction Layer`][System Abstraction Layer] are specific to a platform:

- Allocating / Freeing memory,
- Getting the time,
- Rebooting,
- Sending / Receiving network messages,
- Storing and retrieving the security keys,
- And, so on.

FreeRTOS will be used here to explain the porting of IOWA on a platform.

## FreeRTOS configuration

FreeRTOS has multiple levels to handle the memory allocation:

- Heap 1: The basic memory allocation, does not permit a free,
- Heap 2: Permits memory to be freed, but without coalescence adjacent free blocks,
- Heap 3: Works in the same way as malloc and free and handles thread safety,
- Heap 4: Includes the coalescence adjacent free blocks,
- Heap 5: Adds the ability to span the heap across multiple non-adjacent memory areas.

Since IOWA needs to allocate and deallocate memory, the heap 2 is the minimum requirement. But the recommended heap to use with IOWA is heap 4.

Moreover, the recommended configuration of `FreeRTOSConfig.h` is (not all the defines are provided):

```c
#define configUSE_16_BIT_TICKS           0
#define configUSE_MUTEXES                1
#define configSUPPORT_STATIC_ALLOCATION  0
#define configSUPPORT_DYNAMIC_ALLOCATION 1
#define configTOTAL_HEAP_SIZE            ((size_t)14000)
#define configAPPLICATION_ALLOCATED_HEAP 0

#define INCLUDE_vTaskDelay 1
```

## First step

To begin, create an empty FreeRTOS project with just the main task to start the scheduler:

```c
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"

#define INFINITE_LOOP() while (1);

int main(void)
{
    // Start the scheduler
    vTaskStartScheduler();

    // We should never get here as control is now taken by the scheduler
    INFINITE_LOOP();
}
```

Generally, IOWA needs two threads to have a working LwM2M Client, but depending of the use case, more threads can be added:

- The first thread is dedicated to run IOWA and let it handle the LwM2M / CoAP stacks,
- The other threads are dedicated for the system tasks: measurement, actuator, etc.

You can refer to the part of the documentation [`IOWA Multithread Environment`][IOWA Multithread Environment] to have additional information about how IOWA behaves in a multithread environment.

## Platform abstraction

Before continuing the IOWA platform functions have to be implemented:

```c
typedef struct
{
    SemaphoreHandle_t globalMutex;
} platform_user_data_t;

void initPlatform(platform_user_data_t *platformUserDataP)
{
    platformUserDataP->globalMutex = xSemaphoreCreateMutex();
    xSemaphoreGive(platformUserDataP->globalMutex);
}

void * iowa_system_malloc(size_t size)
{
    return pvPortMalloc(size);
}

void iowa_system_free(void * pointer)
{
    vPortFree(pointer);
}

int32_t iowa_system_gettime(void)
{
    // Platform dependant
}

void iowa_system_reboot(void * userData)
{
    // Platform dependant or can be a stub function
}

void iowa_system_trace(const char * format,
                       va_list varArgs)
{
    // Platform dependant or can be a stub function
}

void * iowa_system_connection_open(iowa_connection_type_t type,
                                   char * hostname,
                                   char * port,
                                   void * userData)
{
    // Platform dependant
}

void iowa_system_connection_close(void * connP,
                                  void * userData)
{
    // Platform dependant
}

int iowa_system_connection_send(void * connP,
                                uint8_t * buffer,
                                size_t length,
                                void * userData )
{
    // Platform dependant
}

int iowa_system_connection_recv(void * connP,
                                uint8_t * buffer,
                                size_t length,
                                void * userData )
{
    // Platform dependant
}

int iowa_system_connection_select(void ** connArray,
                                  size_t connCount,
                                  int32_t timeout,
                                  void * userData )
{
    // Platform dependant
}

void iowa_system_connection_interrupt_select(void * userData)
{
    // Platform dependant
}

void iowa_system_mutex_lock(void * userData)
{
    platform_user_data_t *platformUserDataP;

    platformUserDataP = (platform_user_data_t *)userData;

    xSemaphoreTake(platformUserDataP->globalMutex, portMAX_DELAY);
}

void iowa_system_mutex_unlock(void * userData)
{
    platform_user_data_t *platformUserDataP;

    platformUserDataP = (platform_user_data_t *)userData;

    xSemaphoreGive(platformUserDataP->globalMutex);
}
```

## Tasks management

We can now update the main function to create our two tasks:

```c
typedef struct
{
    SemaphoreHandle_t initMutex;
    iowa_context_t contextP;
    iowa_sensor_t tempSensorId;
} iowa_user_data_t;

void iowa_task(void const* argument)
{
    // Task dedicated for IOWA operations
    iowa_user_data_t *iowaUserDataP;
    platform_user_data_t platformUserData;
    iowa_status_t result;

    initPlatform(&platformUserData);

    iowaUserDataP = (iowa_user_data_t *)argument;

    // Initialize the IOWA context
    iowaUserDataP->contextP = iowa_init(platformUserData);
    if (iowaUserDataP->contextP == NULL)
    {
        INFINITE_LOOP();
    }

    // Initialize the Client context
    result = iowa_client_configure(iowaUserDataP->contextP, "IOWA_Client_FreeRTOS", NULL, NULL);
    if (result != IOWA_COAP_NO_ERROR)
    {
        INFINITE_LOOP();
    }

    // Add a server
    result = iowa_client_add_server(iowaUserDataP->contextP, 1234, "coap://127.0.0.1:5683", 3600, 0, IOWA_SEC_NONE);
    if (result != IOWA_COAP_NO_ERROR)
    {
        INFINITE_LOOP();
    }

    // Add a sensor object
    result = iowa_client_IPSO_add_sensor(iowaUserDataP->contextP, IOWA_IPSO_TEMPERATURE, 24, "Cel", NULL, -20.0, 50.0, &(iowaUserDataP->tempSensorId));
    if (result != IOWA_COAP_NO_ERROR)
    {
        INFINITE_LOOP();
    }

    xSemaphoreGive(iowaUserData->initMutex); // Synchronization

    // Start the IOWA step
    (void)iowa_step(iowaUserDataP->contextP, -1);

    // We should never get here as control is now taken by IOWA
    INFINITE_LOOP();
}

void system_task(void const* argument)
{
    // Task dedicated for system operations
    iowa_user_data_t *iowaUserDataP;

    iowaUserDataP = (iowa_user_data_t *)argument;

    xSemaphoreTake(iowaUserData->initMutex, portMAX_DELAY); // Synchronization

    while (1)
    {
        float newValue;

        newValue = GET_TEMPERATURE_VALUE();

        (void)iowa_client_IPSO_update_value(iowaUserDataP->contextP, iowaUserDataP->tempSensorId, newValue);

        vTaskDelay(5000);
    }
}

int main(void)
{
    iowa_user_data_t iowaUserData;

    // Create the init semaphore
    iowaUserData->initMutex = xSemaphoreCreateMutex();

    // Create the tasks for IOWA
    xTaskCreate((TaskFunction_t)iowa_task, "iowa", 512, &iowaUserData, tskIDLE_PRIORITY+2, NULL);
    xTaskCreate((TaskFunction_t)system_task, "system", 256, &iowaUserData, tskIDLE_PRIORITY+1, NULL);

    // Start the scheduler
    vTaskStartScheduler();

    // We should never get here as control is now taken by the scheduler
    while (1); // Infinite loop
}
```

## Error handling

Now everything is working, error handling have to put in place in case something went wrong:

- Failed to initialize the IOWA stack,
- Failed to connect to a LwM2M Server,
- And, so on.

IOWA can report events to the Application to inform about internal states such as connection to a LwM2M, new observation started or removed, etc.

These events are reported through the callback [`iowa_event_callback_t`](ClientAPI.md#iowa_event_callback_t), this callback is given when calling the API [`iowa_client_configure()`](ClientAPI.md#iowa_client_configure):

```c
void eventCb(iowa_event_t* eventP,
             void * userData,
             iowa_context_t contextP)
{
    switch (eventP->eventType)
    {
    case IOWA_EVENT_REG_FAILED:
        // Remove then re-add the Server in case the Client wasn't able to reach it
        iowa_client_remove_server(contextP, 1234);
        iowa_client_add_server(contextP, 1234, "coap://127.0.0.1:5683", 3600, 0, IOWA_SEC_NONE);
        break;

    default:
        // Do nothing for the other events
        break;
    }
}

void iowa_task(void const* argument)
{
    ...

    result = iowa_client_configure(iowaUserDataP->contextP, "IOWA_Client_FreeRTOS", NULL, eventCb);

    ...
}
```