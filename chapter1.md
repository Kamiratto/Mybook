* [ ] ### 1.**bold**

2._itatic_

3~~.striktrough~~

4.[http://www.baidu.com](http://www.baidu.com)

5![](http://h.hiphotos.baidu.com/image/pic/item/34fae6cd7b899e51601a7b9c40a7d933c9950da5.jpg)6

* 6
* 7





* [ ] 2423

> eqweqwe
>
> ewqe

```java
package sanlogic.vdi.aop;

import java.lang.reflect.Method;
import java.util.Date;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang.StringUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import sanlogic.vdi.common.utils.BrowserUtils;
import sanlogic.vdi.common.utils.ContextHolderUtils;
import sanlogic.vdi.common.utils.oConvertUtils;
import sanlogic.vdi.constants.CommonConstant;
import sanlogic.vdi.sys.dto.LogDto;
import sanlogic.vdi.sys.service.LogService;

/**
 * 消息发送拦截器，拦截各版块中需要发消息的方法
 * 
 * @author HotStrong
 * 
 */
@Component
@Aspect
public class LogAspect {

	@Autowired
	private LogService logService;

	/**
	 * 添加业务逻辑方法切入点
	 */
	@Pointcut("execution(* sanlogic.vdi.sys.service.*.save(..))")
	public void insertServiceCall() {
	}

	/**
	 * 修改业务逻辑方法切入点
	 */
	@Pointcut("execution(* sanlogic.vdi.sys.service.*.save(..))")
	public void updateServiceCall() {
	}

	/**
	 * 删除业务逻辑方法切入点
	 */
	@Pointcut("execution(* sanlogic.vdi.sys.service.*.delete(..))")
	public void deleteFilmCall() {
	}

	/**
	 * 管理员添加操作日志(后置通知)
	 * 
	 * @param joinPoint
	 * @param rtv
	 * @throws Throwable
	 */
	@AfterReturning(value = "insertServiceCall()", returning = "rtv")
	public void insertServiceCallCalls(JoinPoint joinPoint, Object rtv)
			throws Throwable {

		addLog(joinPoint, rtv, "插入");
	}

	/**
	 * 管理员修改操作日志(后置通知)
	 * 
	 * @param joinPoint
	 * @param rtv
	 * @throws Throwable
	 */
	@AfterReturning(value = "updateServiceCall()", argNames = "rtv", returning = "rtv")
	public void updateServiceCallCalls(JoinPoint joinPoint, Object rtv)
			throws Throwable {

		addLog(joinPoint, rtv, "更新");
	}

	/**
	 * 管理员删除影片操作(环绕通知)，使用环绕通知的目的是 在影片被删除前可以先查询出影片信息用于日志记录
	 * 
	 * @param joinPoint
	 * @param rtv
	 * @throws Throwable
	 */
	/*@Around(value = "deleteFilmCall()", argNames = "rtv")
	public Object deleteFilmCallCalls(ProceedingJoinPoint pjp) throws Throwable {

		
		
		
		Object result = null;
		// 环绕通知处理方法
		try {

			// 获取方法参数(被删除的影片id)
			Integer id = (Integer) pjp.getArgs()[0];
			Film obj = null;// 影片对象
			if (id != null) {
				// 删除前先查询出影片对象
				obj = filmService.getFilmById(id);
			}

			// 执行删除影片操作
			result = pjp.proceed();

			if (obj != null) {

				// 创建日志对象
				Log log = new Log();
				log.setUserid(logService.loginUserId());// 用户编号
				log.setCreatedate(new Date());// 操作时间

				StringBuffer msg = new StringBuffer("影片名 : ");
				msg.append(obj.getFname());
				log.setContent(msg.toString());// 操作内容

				log.setOperation("删除");// 操作

				logService.log(log);// 添加日志
			}

		} catch (Exception ex) {
			ex.printStackTrace();
		}

		return result;
	}*/

	/**
	 * 使用Java反射来获取被拦截方法(insert、update)的参数值， 将参数值拼接为操作内容
	 */
	public String adminOptionContent(Object[] args, String mName)
			throws Exception {

		if (args == null) {
			return null;
		}

		StringBuffer rs = new StringBuffer();
		rs.append(mName);
		String className = null;
		int index = 1;
		// 遍历参数对象
		for (Object info : args) {

			// 获取对象类型
			className = info.getClass().getName();
			className = className.substring(className.lastIndexOf(".") + 1);
			rs.append("[参数" + index + "，类型：" + className + "，值：");

			// 获取对象的所有方法
			Method[] methods = info.getClass().getDeclaredMethods();

			// 遍历方法，判断get方法
			for (Method method : methods) {

				String methodName = method.getName();
				// 判断是不是get方法
				if (methodName.indexOf("get") == -1) {// 不是get方法
					continue;// 不处理
				}

				Object rsValue = null;
				try {

					// 调用get方法，获取返回值
					rsValue = method.invoke(info);

					if (rsValue == null) {// 没有返回值
						continue;
					}

				} catch (Exception e) {
					continue;
				}

				// 将值加入内容中
				rs.append("(" + methodName + " : " + rsValue + ")");
			}

			rs.append("]");

			index++;
		}

		return rs.toString();
	}

	/**
	 * @description 添加业务日志通用方法
	 * @return void
	 * @param joinPoint
	 * @param rtv
	 * @param operate
	 * @throws Exception
	 * @update 2013-4-18
	 */
	public void addLog(JoinPoint joinPoint, Object rtv, String operate)
			throws Exception {
		// 获取登录管理员id
		String adminUserId = logService.getCurrentUserID();

		if (StringUtils.isEmpty(adminUserId)) {// 没有管理员登录
			return;
		}

		// 判断参数
		if (joinPoint.getArgs() == null) {// 没有参数
			return;
		}

		// 获取方法名
		String methodName = joinPoint.getSignature().getName();

		// 获取操作内容
		String opContent = adminOptionContent(joinPoint.getArgs(), methodName);

		HttpServletRequest request = ContextHolderUtils.getRequest();
		String broswer = BrowserUtils.checkBrowse(request);
		// 创建日志对象
		LogDto logDto = new LogDto();
		logDto.setUserId(adminUserId);// 设置管理员id
		logDto.setLogContent(opContent);// 操作内容
		logDto.setLogLevel(CommonConstant.Log_Leavel_INFO);// 操作级别
		logDto.setOperateType(CommonConstant.Log_Type_INSERT);// 操作类型
		logDto.setNote(oConvertUtils.getIp());// 操作人员IP
		logDto.setBroswer(broswer);// 操作员浏览器
		logDto.setOperateTime(new Date());// 操作时间

		logService.save(logDto);// 添加日志
	}

}

```



