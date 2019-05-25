## Spring事务提交后执行代码

在日常开发中常常会有这样的场景，在一段业务代码的最后需要发送MQ消息，但是需要事务提交后再执行发送，否则如果MQ消息消费的很快，去库中查询对应的业务数据，因为事务未提交而查询不到，导致代码报错。本文主要探讨应对这样的场景的一种方案。

写法如下：

```java
@Transactional
public void bussiness(){
    // 处理业务
    // ---
    // 下面的代码事务提交后才执行
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            //这里面是业务代码,例如发送mq
            //mqClient.send(mqMessage);
        }
    });
}

```

这段代码很简单，但是却有一个坑，afterCommit方法中不要写新的操作数据库代码，尤其不要写更新数据库的操作，因为事务不会提交，原因可以参考afterCommit方法的文档。如果确实需要在这里面执行事务提交操作，需要开启一个新的事务，写法如下：

```java
public class AnotherClass{
    // 事务的传播机制一定要写成requires_new
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void foo(){
        //业务代码
    }
}
```

## 封装以上两步

如果这类方法比较多的话，则写起来重复性太多，因而，抽象出来如下：
这里改造了[azagorneanu](http://azagorneanu.blogspot.com/2013/06/transaction-synchronization-callbacks.html)的代码:

```java
public interface AfterCommitExecutor extends Executor {
}
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronizationAdapter;
import org.springframework.transaction.support.TransactionSynchronizationManager;
 
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
@Component
public class AfterCommitExecutorImpl extends TransactionSynchronizationAdapter implements AfterCommitExecutor {
    private static final Logger LOGGER = LoggerFactory.getLogger(AfterCommitExecutorImpl.class);
    private static final ThreadLocal<List<Runnable>> RUNNABLES = new ThreadLocal<List<Runnable>>();
    private ExecutorService threadPool = Executors.newFixedThreadPool(5);
 
    @Override
    public void execute(Runnable runnable) {
        LOGGER.info("Submitting new runnable {} to run after commit", runnable);
        if (!TransactionSynchronizationManager.isSynchronizationActive()) {
            LOGGER.info("Transaction synchronization is NOT ACTIVE. Executing right now runnable {}", runnable);
            runnable.run();
            return;
        }
        List<Runnable> threadRunnables = RUNNABLES.get();
        if (threadRunnables == null) {
            threadRunnables = new ArrayList<Runnable>();
            RUNNABLES.set(threadRunnables);
            TransactionSynchronizationManager.registerSynchronization(this);
        }
        threadRunnables.add(runnable);
    }
 
    @Override
    public void afterCommit() {
        List<Runnable> threadRunnables = RUNNABLES.get();
        LOGGER.info("Transaction successfully committed, executing {} runnables", threadRunnables.size());
        for (int i = 0; i < threadRunnables.size(); i++) {
            Runnable runnable = threadRunnables.get(i);
            LOGGER.info("Executing runnable {}", runnable);
            try {
                threadPool.execute(runnable);
            } catch (RuntimeException e) {
                LOGGER.error("Failed to execute runnable " + runnable, e);
            }
        }
    }
 
    @Override
    public void afterCompletion(int status) {
        LOGGER.info("Transaction completed with status {}", status == STATUS_COMMITTED ? "COMMITTED" : "ROLLED_BACK");
        RUNNABLES.remove();
    }
 
}
public void insert(TechBook techBook){
        bookMapper.insert(techBook);
 
//        send after tx commit but is async
//        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
//            @Override
//            public void afterCommit() {
//                executorService.submit(new Runnable() {
//                    @Override
//                    public void run() {
//                        System.out.println("send email after transaction commit...");
//                        try {
//                            Thread.sleep(10*1000);
//                        } catch (InterruptedException e) {
//                            e.printStackTrace();
//                        }
//                        System.out.println("complete send email after transaction commit...");
//                    }
//                });
//            }
//        }
//        );
 
        //send after tx commit and is async
        afterCommitExecutor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(5*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("send email after transactioin commit");
            }
        });
 
//        async work but tx not work, execute even when tx is rollback
//        asyncService.executeAfterTxComplete();
 
        ThreadLocalRandom random = ThreadLocalRandom.current();
        if(random.nextInt() % 2 ==0){
            throw new RuntimeException("test email transaction");
        }
        System.out.println("service end");
    }
```

AOP方式封装

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AfterCommit {}
```

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class AfterCommitAnnotationAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(AfterCommitAnnotationAspect.class);

    private final AfterCommitExecutor afterCommitExecutor;

    @Autowired
    public AfterCommitAnnotationAspect(AfterCommitExecutor afterCommitExecutor) {
        this.afterCommitExecutor = afterCommitExecutor;
    }

    @Around(value = "@annotation(org.zmeu.blog.spring.transaction.AfterCommit)", argNames = "pjp")
    public Object aroundAdvice(final ProceedingJoinPoint pjp) {
        afterCommitExecutor.execute(new PjpAfterCommitRunnable(pjp));
        return null;
    }

    private static final class PjpAfterCommitRunnable implements Runnable {
        private final ProceedingJoinPoint pjp;

        public PjpAfterCommitRunnable(ProceedingJoinPoint pjp) {
            this.pjp = pjp;
        }

        @Override
        public void run() {
            try {
                pjp.proceed();
            } catch (Throwable e) {
                LOGGER.error("Exception while invoking pjp.proceed()", e);
                throw new RuntimeException(e);
            }
        }

        @Override
        public String toString() {
            String typeName = pjp.getTarget().getClass().getSimpleName();
            String methodName = pjp.getSignature().getName();
            return "PjpAfterCommitRunnable[type=" + typeName + ", method=" + methodName + "]";
        }
    }

}
```

