# Spring Batch



## 3.Spring Batch监听器

Spring 支持如下监听器

| **监听器**            | **说明**                                                     |
| --------------------- | ------------------------------------------------------------ |
| JobExecutionListener  | 在 Job 开始前(beforeJob)和之后(afterJob)触发                 |
| StepExecutionListener | 在 Step 开始前(beforeStep)和之后(afterStep)触发              |
| ChunkListener         | 在 Chunk 开始前(beforeChunk)，之后(afterChunk)和错误后(afterChunkError)触发 |
| ItemReadListener      | 在 Read 开始之前(beforeRead)，之后(afterRead)和错误后(onReadError)触发 |
| ItemProcessListener   | 在 Read 开始之前(beforeProcess)，之后(afterProcess)和错误后(onProcessError)触发 |
| ItemWriteListener     | 在 Read 开始之前(beforeWrite)，之后(afterWrite)和错误后(onWriteError)触发 |
| SkipListener          | 在 Read 开始之前(beforeWrite)，之后(afterWrite)和错误后(onWriteError)触发 |



### 3.1 Job监听器

**方式一：JobExecutionListener继承方式**

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;

public class SimpleJobExecutionListener implements JobExecutionListener {
	@Override
	public void beforeJob(JobExecution jobExecution) {
		System.out.println("SimpleJobExecutionListener.beforeJob");
	}

	@Override
	public void afterJob(JobExecution jobExecution) {
		System.out.println("SimpleJobExecutionListener.afterJob");
	}
}
```

**方式二：JobExecutionListener注解方式**

```java
public class JobExecListener {
    @BeforeJob
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("before start time: " + jobExecution.getStartTime());
    }

    @AfterJob
    public void afterJob(JobExecution jobExecution) {
        System.out.println("after end time: " + jobExecution.getEndTime());
    }
}
```

### 3.2 Step监听器

**方式一：StepExecutionListener继承方式**

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.ExitStatus;
import org.springframework.batch.core.StepExecution;
import org.springframework.batch.core.StepExecutionListener;

public class SimpleStepExecutionListener implements StepExecutionListener {
	@Override
	public void beforeStep(StepExecution stepExecution) {
		System.out.println("SimpleStepExecutionListener.beforeStep");
	}

	@Override
	public ExitStatus afterStep(StepExecution stepExecution) {
		System.out.println("SimpleStepExecutionListener.afterStep");
		return stepExecution.getExitStatus();
	}
}
```

**方式二：StepExecutionListener注解方式**

### 3.3 Chunk监听器

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.ChunkListener;
import org.springframework.batch.core.scope.context.ChunkContext;

public class SimpleChunkListener implements ChunkListener {
	@Override
	public void beforeChunk(ChunkContext context) {
		System.out.println("SimpleChunkListener.beforeChunk");
	}

	@Override
	public void afterChunk(ChunkContext context) {
		System.out.println("SimpleChunkListener.afterChunk");
	}

	@Override
	public void afterChunkError(ChunkContext context) {
		System.out.println("SimpleChunkListener.afterChunkError");
	}
}
```

###  3.4 ItemRead监听器

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.ItemReadListener;

public class SimpleItemReadListener implements ItemReadListener<People> {
	@Override
	public void beforeRead() {
		System.out.println("SimpleItemReadListener.beforeRead");
	}

	@Override
	public void afterRead(People item) {
		System.out.println("SimpleItemReadListener.afterRead -- " + item.getName());
	}

	@Override
	public void onReadError(Exception ex) {
		System.out.println("SimpleItemReadListener.onReadError");
	}
}
```

### 3.5 ItemProcess监听器

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.ItemProcessListener;

public class SimpleItemProcessListener implements ItemProcessListener<People, People> {
	@Override
	public void beforeProcess(People item) {
		System.out.println("SimpleItemProcessListener.beforeProcess");
	}

	@Override
	public void afterProcess(People item, People result) {
		System.out.println("SimpleItemProcessListener.afterProcess -- " + result.getName());
	}

	@Override
	public void onProcessError(People item, Exception e) {
		System.out.println("SimpleItemProcessListener.onProcessError");
	}
}
```

### 3.6 ItemWrite监听器

```java
package shangbo.springbatch.example6;
import java.util.List;
import org.springframework.batch.core.ItemWriteListener;

public class SimpleItemWriteListener implements ItemWriteListener<People> {
	@Override
	public void beforeWrite(List<? extends People> items) {
		System.out.println("SimpleItemWriteListener.beforeWrite");
	}

	@Override
	public void afterWrite(List<? extends People> items) {
		System.out.println("SimpleItemWriteListener.afterWrite");
	}

	@Override
	public void onWriteError(Exception exception, List<? extends People> items) {
		System.out.println("SimpleItemWriteListener.onWriteError");
	}
}
```

### 3.7 SkipListener监听器

```java
package shangbo.springbatch.example6;
import org.springframework.batch.core.SkipListener;

public class SimpleSkipListener implements SkipListener<String, People> {
	@Override
	public void onSkipInRead(Throwable t) {
		System.out.println("SimpleSkipListener.onSkipInRead");
	}

	@Override
	public void onSkipInWrite(People item, Throwable t) {
		System.out.println("SimpleSkipListener.onSkipInWrite");
	}

	@Override
	public void onSkipInProcess(String item, Throwable t) {
		System.out.println("SimpleSkipListener.onSkipInProcess");
	}
}
```

## 4.Job参数

###   4.1 job参数

