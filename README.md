package efms.ms.rest.controller;

import java.util.Map;
import java.util.List;
import java.util.ArrayList;
import java.util.HashMap;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import efms.wfm.ms.ajsc.vo.CompleteTaskRequest;
import efms.wfm.ms.ajsc.core.AJSCTaskHandlerImpl;



import org.springframework.http.ResponseEntity;

import efms.log.Log;
import efms.log.LogFactory;
import efms.ms.acl.MSSystemUser;
import efms.ms.automation.ai.ChatBotManager;
import efms.ms.bo.MSIPFlexStackedOrdersManager;
import efms.ms.bo.automation.MSSkipBillingComponent;
import efms.ms.dao.MSBvoipOrderExtensionDao;
import efms.ms.dao.MSTaskRecoveryDao;
import efms.ms.entity.MSFinderException;
import efms.ms.entity.MSJobType;
import efms.ms.op.MSBvoipOrderExtension;
import efms.ms.op.MSIPOrder;
import efms.ms.op.MSIPOrderEntity;
import efms.ms.op.MSIPOrderManager;
import efms.ms.op.MSIPOrderStatus;
import efms.ms.op.MSIPSubOrder;
import efms.ms.op.MSIPSubOrderManager;
import efms.ms.op.MSIPSubOrderTaskManager;
import efms.ms.op.MSIPTask;
import efms.ms.op.MSJob;
import efms.ms.op.MSJobEntity;
import efms.ms.op.MSJobStatus;
import efms.ms.op.MSTaskStatus;
import efms.ms.wfm.MSWFM.Type;
import efms.ms.op.TaskErrorVO;
import efms.ms.tool.MSWFMToolManager;
import efms.ms.wfm.MSWFM;
import efms.ms.wfm.MSWFTask;
import efms.ms.wfm.MSWFTaskHandler;
import efms.ms.wfm.MSWorkFlow;
import efms.ms.wfm.ipmis.MSIPWaitEntity;
import efms.wfm.ms.ajsc.vo.AsyncResultData;
import efms.wfm.ms.dao.TaskErrorDAO;
import efms.wfm.ms.dao.TaskErrorDAOImpl;
import efms.ms.rest.vo.MSTaskChatBotVO;
import efms.ms.op.MSTask;
import efms.ms.op.MSTaskRecovery;
import efms.ms.rest.vo.MSRequestChatBotVO;
import efms.ms.rest.vo.MSTaskActionVO;
import efms.ms.rest.vo.MSChatBotVO;

@RestController
@RequestMapping(value = "/CBController")
public class ChatBotController extends EFMSRestController {

	private static Log _logger = LogFactory.getLog(ChatBotController.class);

	@RequestMapping(value = "/retrigger", method = RequestMethod.POST)
	public @ResponseBody String taskRetrigger(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {
		MSIPSubOrderTaskManager stMgr = new MSIPSubOrderTaskManager(new MSSystemUser());
		/*
		 * if(!stMgr.isProjectEnabled("ChAT_BOT_OPERATION")){ return
		 * "Feature not enabled"; }
		 */

		MSWFMToolManager ms = new MSWFMToolManager();
		String taskid = cb.getTaskid();
		String jobid = cb.getJobid();
		String action = cb.getAction();
		String answer = cb.getAnswer();
		MSWFTaskHandler handler = null;

		try {
			if ("iskip".equalsIgnoreCase(action)) {
				handler = MSWFM.getTaskHandler(taskid);
				MSWFTask t = (MSWFTask) handler.getCurrentTask();
				MSWFM.getTaskHandler(t.getParent().getId()).skip(true);

			} else {
				if ("teretry".equalsIgnoreCase(answer)) {
					cb.setAction("complete");
					cb.setAnswer("Retrigger");
					action = "complete";
				} else if ("teexit".equalsIgnoreCase(answer)) {
					cb.setAction("complete");
					cb.setAnswer("Exit");
					action = "complete";
				}
				
				ms.triggerAction(jobid, taskid, action, answer);
				

				
			}

		} catch (Exception e) {
			_logger.error("Exception in retriggering from ChatBot ", e);
			return "Failed due to " + e.getMessage();
		}
		return "Success";
	}

	@RequestMapping(value = "/taskList", method = RequestMethod.POST)
	public ResponseEntity<List<MSTaskChatBotVO>> taskList(@RequestBody MSRequestChatBotVO rcb,
			@RequestHeader Map<String, String> headers) {
		MSIPSubOrderTaskManager stMgr = new MSIPSubOrderTaskManager(new MSSystemUser());
		/*
		 * if(!stMgr.isProjectEnabled("ChAT_BOT_OPERATION")){ return
		 * "Feature not enabled"; }
		 */
		List<MSTaskChatBotVO> taskList = null;
		boolean status = true;

		MSWFMToolManager ms = new MSWFMToolManager();
		String requestId = rcb.getRequestId();
		int requestType = rcb.getRequestType();

		try {
			taskList = ms.findSystemTasksByEntityTypeAndId(requestType, requestId, true);
		} catch (Exception e) {
			_logger.error("Exception in retriggering from ChatBot ", e);
			status = false;

		}
		return _getJsonresponse(taskList, status);
	}

	@RequestMapping(value = "/action", method = RequestMethod.POST)
	public @ResponseBody String actionPerform(@RequestBody MSChatBotVO msTask,
			@RequestHeader Map<String, String> headers) {

		MSIPSubOrderTaskManager stMgr = new MSIPSubOrderTaskManager(new MSSystemUser());
		MSWFMToolManager ms = new MSWFMToolManager();

		String orderNumber = msTask.getOrderNumber();
		int taskLevel = msTask.getLevel();
		String taskName = msTask.getTaskName();
		String action = msTask.getAction();

		MSIPOrder msiporder = new MSIPOrder();
		int orderid;
		int entityId = 0;
		String jobId = null;
		MSWorkFlow wf;

		List<Integer> entityIdlist = new ArrayList<Integer>();

		try {

			if (MSJobType.ORDER.getCode() == taskLevel) {
				MSIPOrderManager oMgr = new MSIPOrderManager();
				MSIPOrder o = oMgr.findByUsrpOrderId(orderNumber);
				entityId = o.getId();
				entityIdlist.add(entityId);

			} else if (MSJobType.SUBORDER.getCode() == taskLevel) {

				MSIPSubOrderManager soMgr = new MSIPSubOrderManager();
				List<MSIPSubOrder> soList = soMgr.findSubOrdersByUsrpOrderNumber(orderNumber);

				for (MSIPSubOrder so : soList) {
					entityIdlist.add(so.getId());
				}

			} else if (MSJobType.CLIENT_REQUEST.getCode() == taskLevel) {
				return "As of now only for order and sub order level";
			} else if (MSJobType.SERVICE_REQUEST.getCode() == taskLevel) {
				return "As of now only for order and sub order level";
			}

			for (Integer id : entityIdlist) {
				try {
					MSJob j = MSJobEntity.findByEntity(taskLevel, id);
					wf = MSWFM.getWorkFlow(j.getWorkflowJobId());
					_logger.info("work flow job id " + j.getWorkflowJobId());
					List<MSWFTask> taskList = wf.getTasks(taskName);
					for (MSWFTask mswfTask : taskList) {
						_logger.info("work flow task id " + mswfTask.getId());
						if (MSTaskStatus.DONE.getName().equalsIgnoreCase(mswfTask.getStatus())
								&& "retrigger".equalsIgnoreCase(action))
							action = "retrigger";
						else if (MSTaskStatus.READY.getName().equalsIgnoreCase(mswfTask.getStatus())
								&& "retrigger".equalsIgnoreCase(action))
							action = "release";
						_logger.info("work flow job id " + j.getWorkflowJobId() + " task id " + mswfTask.getId()
								+ " action " + action);
						ms.triggerAction(j.getWorkflowJobId(), mswfTask.getId(), action);

						break; // TODO As of now we are retriggering which ever task comes fast no segment
								// validation
					}

				} catch (Exception e) {
					_logger.error("exception = ", e);
					return "fail due to " + e.getMessage();
				}
			}

		} catch (Exception e) {
			_logger.error("Exception in while performing action from ChatBot ", e);

			return "fail due to " + action + " " + e.getMessage();
		}

		return "Success";
	}

	@RequestMapping(value = "/ping", method = RequestMethod.GET)
	public @ResponseBody String ping() {
		_logger.debug("successfully pinged ");
		return "successfully pinged";
	}

	@RequestMapping(value = "/SPPActive/Order/{solutionNumber}", method = RequestMethod.POST)
	public @ResponseBody String getSPPInProgressOrder(@PathVariable String solutionNumber,
			@RequestHeader Map<String, String> headers) {

		MSIPFlexStackedOrdersManager manager = new MSIPFlexStackedOrdersManager();
		MSIPOrder order = null;
		MSBvoipOrderExtension ordExt = null;
		_logger.info("Checking in-progress in SPP for site of order " + solutionNumber);
		String sppInProgressOrder = null;
		String giomSiteId = null;
		int fromDualSiteId = 0;

		try {
			order = MSIPOrderEntity.findByUsrpOrderId(solutionNumber);
			ordExt = MSBvoipOrderExtensionDao.findByUsrpOrderNumber(solutionNumber);
			fromDualSiteId = ordExt.getFromDualSiteId();
			giomSiteId = order.getGiomSiteId();
			sppInProgressOrder = manager.getProvisioningOrder(order);
			_logger.info("SPP in-progess order " + sppInProgressOrder);

		} catch (MSFinderException e) {
			_logger.error("Could not found order ", e);
			return "Order not found in EFMS";
		} catch (Exception e) {
			_logger.error("Error while checking order in SPP ", e);
			return "Unknown error while checking";
		}

		if (sppInProgressOrder != null) {
			if (fromDualSiteId > 0) {
				return sppInProgressOrder + " is in-progress with SPP for GIOM Site id  " + giomSiteId
						+ " from Dual Site Id " + fromDualSiteId;
			} else {
				return sppInProgressOrder + " is in-progress with SPP for GIOM Site id  " + giomSiteId;
			}

		} else {
			if (fromDualSiteId > 0) {
				return " No order in-progress with SPP for GIOM Site id  " + giomSiteId + " from Dual Site Id "
						+ fromDualSiteId;
			} else {
				return " No order in-progress with SPP for GIOM Site id  " + giomSiteId;
			}

		}

	}

	@RequestMapping(value = "/taskdiagnose", method = RequestMethod.POST)
	public @ResponseBody String taskDiagnose(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {

		String orderNumber = null;
		String taskName = null;
		String result = null;

		boolean taskReadyStatus = false;
		MSIPOrder order = null;
		ChatBotManager manager = new ChatBotManager();
		
		try {

				orderNumber = cb.getOrderNumber();
				taskName = cb.getTaskName();
				
				order = MSIPOrderEntity.findByUsrpOrderId(orderNumber);
				taskReadyStatus = verifyTaskStatusInReady(order, taskName);
				
				if(!taskReadyStatus) {
					return taskName+" is not in ready state to investigate.";
				}
				
				
				if("Wait for Inprogress Stacked orders to Complete Gate2".equalsIgnoreCase(taskName)) {
					result = manager.verifyInprogressStackedOrdersToCompleteGate2(order, taskName);
				}else if("Hold FROM SITE IPFR Dual Stack orders".equalsIgnoreCase(taskName)){
					result = manager.verifyHoldFromSiteIPFRDualStackOrders(order, taskName);
				}else if("Wait for IPFlex Parent Orders to Complete Gate1".equalsIgnoreCase(taskName)){
					result = manager.verifyIPFLEXParentOrdersToCompleteGate1(order, taskName);
				}else if("Wait for IPFlex Parent NS Orders to Complete Gate3".equalsIgnoreCase(taskName)){
					result = manager.verifyIPFlexParentNSOrdersToCompleteGate3(order, taskName);
				}else if("Wait for IPFlex Parent NS Orders to Complete Gate2".equalsIgnoreCase(taskName)) {
					result = manager.verifyIPFlexParentNSOrdersToCompleteGate2(order, taskName);
				}else if ("Wait for Parent Orders to Complete Gate1".equalsIgnoreCase(taskName) ||
						"Wait for Parent Orders to Complete Gate2".equalsIgnoreCase(taskName)){
					result = manager.verifyParentOrderToCompleteGate1And2(orderNumber,taskName);
				}else if("Wait for Previous Order to Complete Gate2".equalsIgnoreCase(taskName)){
					result = taskName +" is yet to analyse by efms_wf_chat_bot. Contact admin to get it added.";
				}else {
				
					result = taskName +" is yet to analyse by efms_wf_chat_bot. Contact admin to get it added.";
				}
				
		} catch (Exception e) {
			_logger.error("Error while investigating order ", e);
			result = " Error while analysing "+taskName+" for "+orderNumber+" is :"+e.getMessage();
		}
		
		return result;

	}
	
	@RequestMapping(value = "/falloutTask", method = RequestMethod.POST)
	public @ResponseBody String getFalloutTaskDetails(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {

		ChatBotManager manager = new ChatBotManager();
		TaskErrorDAOImpl teDao = new TaskErrorDAOImpl();
		String taskId = null;
		String result = null;
		String orderNumber = null;
		String taskName = null;
		TaskErrorVO taskError = null;
		String errorCode = null;
		String errorMessage = null;
		List<MSTaskRecovery> taskRecoveryList = null;
		List<MSTaskRecovery> taskRecoveryList2 = null;
		MSTaskRecoveryDao taskRecoveryDao = new MSTaskRecoveryDao();

		try {
			orderNumber = cb.getOrderNumber();
			taskName = cb.getTaskName();
			_logger.info("getFalloutTaskDetails for order: " + orderNumber + " task: " + taskName);
			taskId = getReadyTaskId(orderNumber, taskName);
			if (taskId == null) {
				result = taskName + " is not in ready state to provide details";
			} else {
				taskError = teDao.getTaskErrorVO(taskId);

				errorCode = taskError.getErrorCode();
				errorMessage = taskError.getErrorMessage();

				try {
					taskRecoveryList = taskRecoveryDao.findByErrorCode(errorCode);
					taskRecoveryList2 = taskRecoveryDao.findByErrorMessage(errorMessage);
					taskRecoveryList.addAll(taskRecoveryList2);
				} catch (MSFinderException e) {
					_logger.error("Couldn't find the error details  in ms_task_recovery", e);
					taskRecoveryList = taskRecoveryDao.findByErrorMessage(errorMessage);

				}

				if (taskRecoveryList != null && !taskRecoveryList.isEmpty()) {
					for (MSTaskRecovery recovery : taskRecoveryList) {
						if (recovery.getErrorMessage() != null && errorMessage != null) {
							if (errorMessage.contains(recovery.getErrorMessage())) { 
								result = null;
								result = "<b>Error Code :</b> " + errorCode + " <br><b>Error Message :</b> " + errorMessage +" <br><b>Errored System :</b> "+recovery.getErrorSystem()
										+ " <br><b>Possible solution :</b> " + recovery.getSolution();
								break;
							}
						} 
					}
					
					if(result == null) {
						for (MSTaskRecovery recovery : taskRecoveryList) {
							if(taskName.equalsIgnoreCase(recovery.getFalloutTaskName()) && recovery.getErrorMessage() == null
									&& errorCode != null && errorCode.equalsIgnoreCase(recovery.getErrorCode())){ // As SPP has all same error code
								result = "<b>Error Code :</b> " + errorCode + " <br><b>Error Message :</b> " + errorMessage +" <br><b>Errored System :</b> "+recovery.getErrorSystem()
								+ " <br><b>Possible solution :</b> " + recovery.getSolution();
								break;
							}
						}
					}
					
					
					
				} else {
					result = "<b>Error Code :</b> " + errorCode + " <br><b>Error Message :</b> " + errorMessage
							+ " <br><b>Possible solution :</b> NA";
				}

				if (result == null) {
					if(errorCode == null && errorMessage == null) {
						result = "Couldn't find the error message/error code associated with " + taskName;
					}else {
						result = "<b>Error Code :</b> " + errorCode + " <br><b>Error Message :</b> " + errorMessage
								+ " <br><b>Possible solution :</b> NA";
					}
				}
			}

		} catch (Exception e) {

			_logger.error("Couldn't find the error details ", e);
			result = "Failed to find the error message/error code associated with " + taskName;
		}

		return result;

	}
	
	@RequestMapping(value = "/readyTasks", method = RequestMethod.POST)
	public @ResponseBody String getReadyTaskList(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {
		
		String orderNumber = null;
		String resultString = null;
		int entityId = -1;
		MSIPSubOrder subOrder = null;
		MSIPSubOrderManager subOrderManager = new  MSIPSubOrderManager();
		MSIPOrder ord = null;
		MSIPOrderManager ordManager = new MSIPOrderManager();
		List<MSJob> jobList = null;
		int entityType = -1;
		String jobId = null;
		List<MSIPTask> readyTaskList = null;
		
		try {
				orderNumber = cb.getOrderNumber();
				
				_logger.debug("orderNumber  "+orderNumber);
				
				subOrder = subOrderManager.findByServiceOrderBusinessKey(orderNumber);
				
				if(subOrder == null) {
					try {
							ord = ordManager.findByServiceOrderBusinessKey(orderNumber);
						
					}catch(MSFinderException e) {
						return "Order Not Found in EFMS";
					}
					entityId = ord.getId();
					entityType =MSJobType.ORDER.getCode();
							
				}else {
					entityId = subOrder.getId();
					entityType =MSJobType.SUBORDER.getCode();
				}
				
				_logger.debug("entityType  "+entityType+" entityId "+entityId);
				jobList = MSJobEntity.findJobListByEntity(entityType, entityId);
				if( jobList == null || jobList.isEmpty()) {
					return "Workflow is not yet created for GIOM Order Number: "+orderNumber;
				}
				
				for(MSJob j : jobList) {
					if(MSJobStatus.COMPLETED.getCode() == j.getStatus().getCode() ||
							MSJobStatus.CANCELED.getCode() == j.getStatus().getCode()) {
						continue;
					}
					jobId = j.getWorkflowJobId();
					break;
					
				}
				
				readyTaskList = MSIPWaitEntity.getReadyTasksInAllFlows(jobId); 
				
				_logger.debug("readyTaskList  "+readyTaskList);
				if( readyTaskList == null || readyTaskList.isEmpty()) {
					resultString = "No Ready task associated with the order";
					ord = ordManager.findByServiceOrderBusinessKey(orderNumber);
					if(MSIPOrderStatus.COMPLETED == ord.getOrderStatus()) {
						resultString = "Order is already completed in EFMS";
					}else if(MSIPOrderStatus.CANCELED == ord.getOrderStatus()) {
						resultString = "Order is already Cancelled in EFMS";
					}else if(MSIPOrderStatus.CANCELLED_AND_CLONED == ord.getOrderStatus()) {
						resultString = "Order is already Cancelled and Cloned in EFMS";
					}else {
						resultString = "No Ready task associated with the order";
					}
				}else {
					for(MSIPTask task : readyTaskList) {
						if(resultString == null) {
							resultString = task.getName()+"<br> ";
						}else {
							resultString = resultString+task.getName()+"<br> ";
						}
						
					}
					
				}
		}catch(Exception e) {
			_logger.error("Error while getting  ready tasks ", e);
			resultString = "Failed to find the ready tasks associated with " + orderNumber;
		}
		return resultString;
	}
	
	
	@RequestMapping(value = "/switchBilling", method = RequestMethod.POST)
	public @ResponseBody String switchBillingFlow(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {
		
		String resultString = null;
		String orderNumber  = cb.getOrderNumber();
		String action = cb.getAction();
		ChatBotManager manager = new ChatBotManager();
		String userId = cb.getUserId();
		_logger.info("orderNumber "+orderNumber+" action "+action+" userId "+userId);
		
		try {
			resultString = manager.switchBillingFlow(orderNumber, action,userId);
		}catch(Exception e){
			_logger.error("Error while switching billing ",e);
			resultString  = "Failed to switch billing for order "+orderNumber;
		}
		
		return resultString;
		
	}
	public boolean verifyTaskStatusInReady(MSIPOrder order, String taskName) {
		
		String taskId = null;
		
		try{
				taskId = MSIPWaitEntity.findTaskid(order.getUsrpOrderId(), taskName);
				if(taskId != null) {
					return true;
				}
			
		}catch(Exception e) {
			_logger.error("failed to find the ready task ",e);
		}
		
		return false;
		
	}
	
	public String getReadyTaskId(String  orderNumber, String taskName) {
		
		String taskId = null;
		
		try{
				taskId = MSIPWaitEntity.findTaskid(orderNumber, taskName);
				
			
		}catch(Exception e) {
			_logger.error("failed to find the ready task ",e);
		}
		
		return taskId;
		
	}
	
	@RequestMapping(value = "/taskAction", method = RequestMethod.POST)
	public @ResponseBody String taskAction(@RequestBody MSTaskActionVO actionVo,
			@RequestHeader Map<String, String> headers) {
		
		
		String action = null;
		String taskId = null;
		HashMap<String, String> inVariables = null;
		String jobId = null;
		String answer = null;
		
		MSWFMToolManager ms = new MSWFMToolManager();
		MSWFTaskHandler handler = null;
		String completionVariable = null;
		String completionStatus = null;
	
		try {
				action = actionVo.getAction();
				taskId = actionVo.getTaskId();
				inVariables = actionVo.getVariables();
				
				MSWorkFlow msworkflow;
		        MSWFTask mswftask;
		        
		        if(inVariables != null) {
		        	 _logger.info("taskId : "+taskId +" action "+action+" inVariables : "+inVariables.toString());
		        }else {
		        	 _logger.info("taskId : "+taskId +" action "+action+" inVariables is null ");
		        }
		       
		        
				if("complete".equalsIgnoreCase(action)) {
					if(inVariables == null || inVariables.isEmpty()) {
						ms.triggerAction(jobId, taskId, action, answer);
					}else {
						 if (MSWFM.getSource(taskId, Type.TASK) == MSWFM.WFSource.AJSC6) {
							 
							 msworkflow = MSWFM.getWorkFlowByTaskId(taskId);
							 mswftask = msworkflow.findTaskById(taskId);
							 
							 AJSCTaskHandlerImpl ajscHhandler = ( AJSCTaskHandlerImpl ) MSWFM.getTaskHandler(msworkflow, mswftask);

			                    AsyncResultData a = new AsyncResultData();
			                    Map<String, String> variables = new HashMap<String, String>();
			                    a.setVariables( inVariables );
			                    completionStatus ="success"; //modify if needed
			                    a.setResult( completionStatus );

			                    CompleteTaskRequest completeReq = new CompleteTaskRequest();
			                    completeReq.setTaskId( taskId );
			                    completeReq.setAsyncResultData(a);

			                    ajscHhandler.complete("Completing task from EFMS Bot", completeReq );

						 }
					}
				}else if("skip".equalsIgnoreCase(action) ||
						"retrigger".equalsIgnoreCase(action) ||
						"release".equalsIgnoreCase(action)) {
					ms.triggerAction(jobId, taskId, action, answer);
				}else if("iskip".equalsIgnoreCase(action)) {
					handler = MSWFM.getTaskHandler(taskId);
					MSWFTask t = (MSWFTask) handler.getCurrentTask();
					MSWFM.getTaskHandler(t.getParent().getId()).skip(true);
				}else if ("teretry".equalsIgnoreCase(action)) {
					action = "complete";
					answer = "Retrigger";
					ms.triggerAction(jobId, taskId, action, answer);
				} else if ("teexit".equalsIgnoreCase(action)) {
					action = "complete";
					answer = "Exit";
					ms.triggerAction(jobId, taskId, action, answer);
				}else {
					return "Operation Not Supported by BOT. Please check with Admin.";
				}
				
				
			
		}catch(Exception e) {
			_logger.error("Error while performing taskAction ",e);
			return "Failed to perform action";
		}
		
		return "Success";
	}
	
	@RequestMapping(value = "/skipBp3Billing", method = RequestMethod.POST)
	public @ResponseBody String skipBp3Billing(@RequestBody MSTaskChatBotVO cb,
			@RequestHeader Map<String, String> headers) {
		
		String resultString = null;
		String orderNumber  = cb.getOrderNumber();
		String userId = cb.getUserId();
		_logger.info("orderNumber "+orderNumber+" userId "+userId);
		MSSkipBillingComponent comp = new MSSkipBillingComponent();
		
		try {
			resultString = comp.processSkipping(orderNumber);
		}catch(Exception e){
			_logger.error("Error in skipBp3Billing ",e);
			resultString  = "Operation Failed "+orderNumber;
		}
		
		return resultString;
		
	}

	public static void main(String args[]) {
		try {
			ChatBotController cbc = new ChatBotController();
			/*
			 * MSTaskChatBotVO msTaskChatBotVO = new MSTaskChatBotVO();
			 * msTaskChatBotVO.setTaskid(args[1]); msTaskChatBotVO.setJobid(args[0]);
			 */

			Map<String, String> headers = null;
			// System.out.println("wf input - "+msTaskChatBotVO.toString());

			// String output = cbc.taskRetrigger(msTaskChatBotVO, headers );
			// System.out.println("Retrigger done "+output);

			MSRequestChatBotVO mSRequestChatBotVO = new MSRequestChatBotVO();
			mSRequestChatBotVO.setRequestId(args[0]);
			mSRequestChatBotVO.setRequestType(5);
			ResponseEntity<List<MSTaskChatBotVO>> taskLists = cbc.taskList(mSRequestChatBotVO, headers);
			System.out.println("the response is = " + taskLists);
			// for(MSTask msTask : taskLists )
			// System.out.println("Name "+msTask.getName());

			System.out.println("List done ");

		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			System.exit(0);
		}

	}
}
