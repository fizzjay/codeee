using System;
using System.Collections;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.IdentityModel.Tokens.Jwt;
using System.Linq;
using System.Net.WebSockets;
using System.Reflection;
using System.Runtime.CompilerServices;
using System.Runtime.Serialization;
using System.Text;
using System.Text.RegularExpressions;
using System.Threading;
using System.Threading.Tasks;
using ATT.PlayerStates;
using ATT.TownshipServer;
using Alta.Alchemy;
using Alta.Api.Client.HighLevel;
using Alta.Api.DataTransferModels.Models.Requests;
using Alta.Api.DataTransferModels.Models.Responses;
using Alta.Api.DataTransferModels.Models.Shared;
using Alta.Api.DataTransferModels.Utility;
using Alta.Character;
using Alta.Console;
using Alta.Global;
using Alta.Impact;
using Alta.Inventory;
using Alta.Lighting;
using Alta.Map;
using Alta.Networking;
using Alta.Networking.Scripts.Player;
using Alta.QuickAccessActions;
using Alta.ScreenshotCamera;
using Alta.Serialization;
using Alta.StatSystem;
using Alta.Static;
using Alta.Trading;
using Alta.UserAuthentication;
using Alta.Utilities;
using Alta.Voice;
using Basemodal;
using CrossGameplayApi;
using Features.Climbing;
using HarmonyLib;
using MelonLoader;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using Township.Downed;
using UnifiedMod.Api;
using UnifiedMod.Controls;
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.Controls;
using UnityEngine.Rendering;
using UnityEngine.SceneManagement;
using UnityEngine.UI;
using VivoxUnity;

namespace UnifiedMod;

public class AdonaiUnifiedMod : MelonMod
{
	internal static class Pc2qPatch
	{
		[HarmonyPatch(typeof(JoinServerPipeline), "ConnectionApprovedByServer")]
		private class ConnectionApprovedPatch
		{
			[CompilerGenerated]
			private static class _003C_003EO
			{
				public static ConnectionEventHandler _003C0_003E__ConnectionToServerLost;
			}

			private unsafe static bool Prefix(JoinServerPipeline __instance, ref Connection connection)
			{
				//IL_002d: Unknown result type (might be due to invalid IL or missing references)
				//IL_0033: Expected O, but got Unknown
				//IL_00b5: Unknown result type (might be due to invalid IL or missing references)
				//IL_00bc: Expected O, but got Unknown
				//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
				//IL_00d5: Expected O, but got Unknown
				//IL_010c: Unknown result type (might be due to invalid IL or missing references)
				//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
				//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
				//IL_00fc: Expected O, but got Unknown
				//IL_0142: Unknown result type (might be due to invalid IL or missing references)
				//IL_0147: Unknown result type (might be due to invalid IL or missing references)
				//IL_0248: Unknown result type (might be due to invalid IL or missing references)
				//IL_0252: Unknown result type (might be due to invalid IL or missing references)
				//IL_02bd: Unknown result type (might be due to invalid IL or missing references)
				//IL_02c4: Expected O, but got Unknown
				//IL_031b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0325: Expected O, but got Unknown
				//IL_0301: Unknown result type (might be due to invalid IL or missing references)
				//IL_030b: Expected O, but got Unknown
				Con = connection;
				MelonLogger.Msg("connectionapproved.");
				ServerJoinResult val = (ServerJoinResult)AccessTools.Field(((object)__instance).GetType(), "joinResult").GetValue(__instance);
				int num = 0;
				try
				{
					num = ((val != null) ? val.ServerIdentifier : 0);
				}
				catch
				{
				}
				if (num > 0 && !IsServerAllowed(num))
				{
					QuitForDisallowed("JoinServerPipeline id=" + num);
					return false;
				}
				MelonLogger.Msg("getting private methods");
				MethodInfo method = AccessTools.Method(typeof(JoinServerPipeline), "ConnectionSequence", (Type[])null, (Type[])null);
				MethodInfo method2 = AccessTools.Method(typeof(JoinServerPipeline), "HandleServerConnectionResponse", (Type[])null, (Type[])null);
				SerializeConnectionMethod val2 = (SerializeConnectionMethod)Delegate.CreateDelegate(typeof(SerializeConnectionMethod), __instance, method);
				SerializeConnectionMethod val3 = (SerializeConnectionMethod)Delegate.CreateDelegate(typeof(SerializeConnectionMethod), __instance, method2);
				MelonLogger.Msg("Connected to the server successfully");
				Connection obj2 = connection;
				object obj3 = _003C_003EO._003C0_003E__ConnectionToServerLost;
				if (obj3 == null)
				{
					ConnectionEventHandler val4 = ConnectionToServerLost;
					_003C_003EO._003C0_003E__ConnectionToServerLost = val4;
					obj3 = (object)val4;
				}
				obj2.Disconnected += (ConnectionEventHandler)obj3;
				MelonLogger.Msg("subscribed event");
				PlatformTarget val5 = (PlatformTarget)2;
				if (!UnifiedMod.Api.main_values.pc2q)
				{
					MelonLogger.Msg("not using pc to quest and setting the platform target to pc");
				}
				int identifier = ((UserInfo)ApiAccess.ApiClient.UserClient.LoggedInUserInfo).Identifier;
				string text = JwtTokenHandler.Write(val.Token);
				PlayerMode playerMode = PlayerModeUtilities.PlayerMode;
				string text2 = ((object)BuildVersion.GetCurrentVersion()).ToString();
				MelonLogger.Msg("joining server with token: " + text + " and platform target: " + ((object)(*(PlatformTarget*)(&val5))/*cast due to .constrained prefix*/).ToString() + " and version: " + text2);
				Type type = AccessTools.TypeByName("Alta.Networking.RequestJoinMessage") ?? AccessTools.TypeByName("RequestJoinMessage");
				if (type == null)
				{
					MelonLogger.Error("[Pc2qPatch] RequestJoinMessage type not found.");
					return true;
				}
				ConstructorInfo constructorInfo = AccessTools.Constructor(type, new Type[5]
				{
					typeof(int),
					typeof(string),
					typeof(PlayerMode),
					typeof(PlatformTarget),
					typeof(string)
				}, false);
				if (constructorInfo == null)
				{
					MelonLogger.Error("[Pc2qPatch] RequestJoinMessage ctor not found.");
					return true;
				}
				object reqObj = constructorInfo.Invoke(new object[5] { identifier, text, playerMode, val5, text2 });
				MethodInfo serializeMi = AccessTools.Method(type, "Serialize", new Type[2]
				{
					typeof(Connection),
					typeof(Stream)
				}, (Type[])null);
				if (serializeMi == null)
				{
					MelonLogger.Error("[Pc2qPatch] RequestJoinMessage.Serialize not found.");
					return true;
				}
				SerializeConnectionMethod val6 = (SerializeConnectionMethod)delegate(Connection c, Stream s)
				{
					serializeMi.Invoke(reqObj, new object[2] { c, s });
				};
				MelonLogger.Msg("sending join message");
				connection.Send((ConnectionChannel)null, (MessageType)9, val6);
				MelonLogger.Msg("setting up handlers");
				connection.SetHandler((MessageType)7, val2);
				connection.SetHandler((MessageType)10, val3);
				if (CacheManager.Instance == null)
				{
					CacheManager.Instance = (CacheManager)new ClientCacheManager();
				}
				Connection obj4 = connection;
				CacheManager instance = CacheManager.Instance;
				obj4.SetHandler((MessageType)26, new SerializeConnectionMethod(instance.HandleCacheRequest));
				MelonLogger.Msg("finished connection approved");
				return false;
			}

			private static void ConnectionToServerLost(Connection connection)
			{
				MelonLogger.Msg("Connection to server lost");
				UnifiedMod.Api.main_values.is_connected = false;
			}
		}

		public static Connection Con;
	}

	private class StakeDrawDriver : MonoBehaviour
	{
		private void Update()
		{
			PancakesBehaviour.HandleContinuousDraw();
			if (Keyboard.current != null && ((ButtonControl)Keyboard.current.escapeKey).wasPressedThisFrame && PancakesBehaviour.drawModeActive)
			{
				PancakesBehaviour.InvokeExit();
			}
		}
	}

	private static class PancakesBehaviour
	{
		private class DrawTarget
		{
			public Vector3 p;

			public Vector3 n;

			public DrawTarget(Vector3 p, Vector3 n)
			{
				//IL_0007: Unknown result type (might be due to invalid IL or missing references)
				//IL_0008: Unknown result type (might be due to invalid IL or missing references)
				//IL_000e: Unknown result type (might be due to invalid IL or missing references)
				//IL_000f: Unknown result type (might be due to invalid IL or missing references)
				this.p = p;
				this.n = n;
			}
		}

		public static bool drawModeActive = false;

		public static Camera drawCam;

		public static Camera mainCamCache;

		public static float drawStandbyBackstep = 3f;

		public static float drawSpacing = 0.3f;

		private static readonly Queue<DrawTarget> drawQueue = new Queue<DrawTarget>();

		private static bool drawWorkerRunning = false;

		private static Vector3 lastPlacedPoint;

		private static bool lastPlacedValid = false;

		private static bool drawMidPlace = false;

		private static int drawQueueMax = 1;

		private static float nextDrawAllowedTime = 0f;

		private static float drawMinInterval = 0.02f;

		public static float playerBackstep = 0.65f;

		public static float handHeightAboveGround = 0.15f;

		public static float handTiltDegrees = 0f;

		public static bool useRightHandForStakes = true;

		private static float rayHeight = 2000f;

		private static int groundMask = -5;

		private static GameObject runnerObj;

		private static StakeDrawDriver driver;

		private static Action _onExit;

		public static Vector3 drawStandbyPos;

		public static Quaternion drawStandbyRot;

		private static bool outlineRoutineRunning = false;

		private static Pickup _heldStake;

		public static void SetExit(Action a)
		{
			_onExit = a;
		}

		public static void InvokeExit()
		{
			try
			{
				_onExit?.Invoke();
			}
			catch
			{
			}
		}

		public static void HandleContinuousDraw()
		{
			//IL_000c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0017: Expected O, but got Unknown
			//IL_0065: Unknown result type (might be due to invalid IL or missing references)
			//IL_0093: Unknown result type (might be due to invalid IL or missing references)
			//IL_0094: Unknown result type (might be due to invalid IL or missing references)
			//IL_007c: Unknown result type (might be due to invalid IL or missing references)
			//IL_007d: Unknown result type (might be due to invalid IL or missing references)
			if (!drawModeActive || (Object)drawCam == (Object)null || Mouse.current == null)
			{
				return;
			}
			if (Mouse.current.leftButton.isPressed && !drawMidPlace && drawQueue.Count < drawQueueMax && Time.time >= nextDrawAllowedTime && TryRayFromCameraFiltered(drawCam, ((InputControl<Vector2>)(object)((Pointer)Mouse.current).position).ReadValue(), out var hitPoint, out var hitNormal) && (!lastPlacedValid || Vector3.Distance(hitPoint, lastPlacedPoint) >= drawSpacing))
			{
				drawQueue.Enqueue(new DrawTarget(hitPoint, hitNormal));
				if (!drawWorkerRunning)
				{
					MelonCoroutines.Start(DrawQueueWorker());
				}
			}
			if (Mouse.current.leftButton.wasReleasedThisFrame)
			{
				nextDrawAllowedTime = 0f;
			}
		}

		private static IEnumerator DrawQueueWorker()
		{
			drawWorkerRunning = true;
			PlayerController pc = PlayerController.Current;
			if ((Object)pc == (Object)null)
			{
				drawWorkerRunning = false;
				yield break;
			}
			PlayerLocomotionController loco = ((Component)pc).GetComponent<PlayerLocomotionController>();
			while (drawModeActive)
			{
				if (drawQueue.Count == 0)
				{
					yield return null;
					continue;
				}
				DrawTarget tgt = drawQueue.Dequeue();
				drawMidPlace = true;
				if (lastPlacedValid && Vector3.Distance(tgt.p, lastPlacedPoint) < drawSpacing * 0.6f)
				{
					drawMidPlace = false;
					nextDrawAllowedTime = Time.time + drawMinInterval;
					yield return null;
					continue;
				}
				TeleportPlayerTo(tgt.p, tgt.n);
				yield return null;
				PoseHandForStake(GetStakeHand(), cam: ((Object)pc.Camera != (Object)null) ? ((Component)pc.Camera).transform : null, groundPoint: tgt.p, normal: tgt.n);
				GrabStake();
				yield return (object)new WaitForSeconds(0.05f);
				DropStakeAt(tgt.p, tgt.n);
				yield return (object)new WaitForSeconds(0.05f);
				lastPlacedPoint = tgt.p;
				lastPlacedValid = true;
				nextDrawAllowedTime = Time.time + drawMinInterval;
				if ((Object)loco != (Object)null)
				{
					((LocomotionController)loco).MoveTo(drawStandbyPos, (LocomotionFunction)0);
				}
				else
				{
					((Component)pc).transform.position = drawStandbyPos;
				}
				drawMidPlace = false;
				yield return null;
			}
			drawWorkerRunning = false;
		}

		public static void ToggleDrawModeOn(AdonaiUnifiedMod owner)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_004b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0056: Expected O, but got Unknown
			//IL_00bb: Unknown result type (might be due to invalid IL or missing references)
			//IL_00c6: Expected O, but got Unknown
			//IL_00dd: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e2: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e7: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f4: Unknown result type (might be due to invalid IL or missing references)
			//IL_011d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0122: Unknown result type (might be due to invalid IL or missing references)
			//IL_0128: Unknown result type (might be due to invalid IL or missing references)
			//IL_012d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0132: Unknown result type (might be due to invalid IL or missing references)
			//IL_013d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0142: Unknown result type (might be due to invalid IL or missing references)
			//IL_0151: Unknown result type (might be due to invalid IL or missing references)
			//IL_015c: Expected O, but got Unknown
			//IL_010d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0112: Unknown result type (might be due to invalid IL or missing references)
			//IL_0173: Unknown result type (might be due to invalid IL or missing references)
			//IL_0160: Unknown result type (might be due to invalid IL or missing references)
			//IL_0199: Unknown result type (might be due to invalid IL or missing references)
			//IL_01a4: Expected O, but got Unknown
			//IL_01da: Unknown result type (might be due to invalid IL or missing references)
			//IL_01e5: Expected O, but got Unknown
			//IL_01ab: Unknown result type (might be due to invalid IL or missing references)
			//IL_01b5: Expected O, but got Unknown
			//IL_01ba: Unknown result type (might be due to invalid IL or missing references)
			//IL_01c4: Expected O, but got Unknown
			//IL_009f: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a5: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			if ((Object)current == (Object)null)
			{
				return;
			}
			lastPlacedValid = false;
			drawMidPlace = false;
			nextDrawAllowedTime = 0f;
			drawQueue.Clear();
			drawWorkerRunning = false;
			mainCamCache = current.Camera;
			if ((Object)mainCamCache == (Object)null)
			{
				MelonLogger.Warning("[DrawMode] No main camera found.");
				return;
			}
			Camera val = null;
			try
			{
				object obj = AccessTools.Field(typeof(AdonaiUnifiedMod), "_pfv3EditorCam")?.GetValue(owner);
				val = (Camera)(((obj is Camera) ? obj : null) ?? mainCamCache);
			}
			catch
			{
				val = mainCamCache;
			}
			drawCam = val;
			if ((Object)drawCam == (Object)null)
			{
				MelonLogger.Warning("[DrawMode] No PFV3 editor camera available.");
				return;
			}
			Vector3 val2 = Vector3.ProjectOnPlane(((Component)drawCam).transform.forward, Vector3.up);
			Vector3 val3 = ((Vector3)(ref val2)).normalized;
			if (((Vector3)(ref val3)).sqrMagnitude < 1E-06f)
			{
				val3 = ((Component)drawCam).transform.forward;
			}
			drawStandbyPos = ((Component)drawCam).transform.position - val3 * drawStandbyBackstep;
			drawStandbyRot = ((Component)current).transform.rotation;
			PlayerLocomotionController component = ((Component)current).GetComponent<PlayerLocomotionController>();
			if ((Object)component != (Object)null)
			{
				((LocomotionController)component).MoveTo(drawStandbyPos, (LocomotionFunction)0);
			}
			else
			{
				((Component)current).transform.position = drawStandbyPos;
			}
			drawModeActive = true;
			SetExit(owner.PFV3_ToggleStakeDraw);
			if ((Object)runnerObj == (Object)null)
			{
				runnerObj = new GameObject("StakeDrawDriver");
				Object.DontDestroyOnLoad((Object)runnerObj);
				driver = runnerObj.AddComponent<StakeDrawDriver>();
			}
			else if ((Object)driver == (Object)null)
			{
				driver = runnerObj.AddComponent<StakeDrawDriver>();
			}
			MelonLogger.Msg("[DrawMode] Enabled.");
		}

		public static void ToggleDrawModeOff()
		{
			//IL_0031: Unknown result type (might be due to invalid IL or missing references)
			//IL_003c: Expected O, but got Unknown
			//IL_0054: Unknown result type (might be due to invalid IL or missing references)
			//IL_005f: Expected O, but got Unknown
			//IL_0080: Unknown result type (might be due to invalid IL or missing references)
			//IL_008b: Expected O, but got Unknown
			//IL_0066: Unknown result type (might be due to invalid IL or missing references)
			//IL_0070: Expected O, but got Unknown
			//IL_0092: Unknown result type (might be due to invalid IL or missing references)
			//IL_009c: Expected O, but got Unknown
			lastPlacedValid = false;
			drawMidPlace = false;
			nextDrawAllowedTime = 0f;
			drawQueue.Clear();
			drawWorkerRunning = false;
			drawCam = null;
			if ((Object)mainCamCache != (Object)null)
			{
				((Behaviour)mainCamCache).enabled = true;
			}
			mainCamCache = null;
			if ((Object)driver != (Object)null)
			{
				try
				{
					Object.Destroy((Object)driver);
				}
				catch
				{
				}
				driver = null;
			}
			if ((Object)runnerObj != (Object)null)
			{
				try
				{
					Object.Destroy((Object)runnerObj);
				}
				catch
				{
				}
				runnerObj = null;
			}
			drawModeActive = false;
			SetExit(null);
			MelonLogger.Msg("[DrawMode] Disabled.");
		}

		private static bool IsSelf(Transform t)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_001f: Unknown result type (might be due to invalid IL or missing references)
			//IL_002a: Expected O, but got Unknown
			//IL_002d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0038: Expected O, but got Unknown
			//IL_003b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0041: Unknown result type (might be due to invalid IL or missing references)
			//IL_004b: Expected O, but got Unknown
			//IL_004b: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			Transform val = (((Object)current != (Object)null) ? ((Component)current).transform : null);
			if (!((Object)val == (Object)null) && !((Object)t == (Object)null))
			{
				if (!((Object)t == (Object)val))
				{
					return t.IsChildOf(val);
				}
				return true;
			}
			return false;
		}

		private static bool TryRayFromCameraFiltered(Camera cam, Vector2 screenPos, out Vector3 hitPoint, out Vector3 hitNormal)
		{
			//IL_0001: Unknown result type (might be due to invalid IL or missing references)
			//IL_0008: Unknown result type (might be due to invalid IL or missing references)
			//IL_000d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0013: Unknown result type (might be due to invalid IL or missing references)
			//IL_001e: Expected O, but got Unknown
			//IL_0023: Unknown result type (might be due to invalid IL or missing references)
			//IL_0024: Unknown result type (might be due to invalid IL or missing references)
			//IL_0029: Unknown result type (might be due to invalid IL or missing references)
			//IL_006a: Unknown result type (might be due to invalid IL or missing references)
			//IL_006f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0077: Unknown result type (might be due to invalid IL or missing references)
			//IL_0082: Expected O, but got Unknown
			//IL_0095: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a0: Expected O, but got Unknown
			//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
			//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
			//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
			//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
			hitPoint = default(Vector3);
			hitNormal = Vector3.up;
			if ((Object)cam == (Object)null)
			{
				return false;
			}
			RaycastHit[] array = Physics.RaycastAll(cam.ScreenPointToRay(Vector2.op_Implicit(screenPos)), 1000f, groundMask, (QueryTriggerInteraction)1);
			Array.Sort(array, (RaycastHit a, RaycastHit b) => ((RaycastHit)(ref a)).distance.CompareTo(((RaycastHit)(ref b)).distance));
			RaycastHit[] array2 = array;
			for (int num = 0; num < array2.Length; num++)
			{
				RaycastHit val = array2[num];
				Transform val2 = (((Object)((RaycastHit)(ref val)).collider != (Object)null) ? ((Component)((RaycastHit)(ref val)).collider).transform : null);
				if (!((Object)val2 == (Object)null) && !IsSelf(val2) && (((Object)val2).name == null || !((Object)val2).name.Contains("Wooden Stake")))
				{
					hitPoint = ((RaycastHit)(ref val)).point;
					hitNormal = ((RaycastHit)(ref val)).normal;
					return true;
				}
			}
			return false;
		}

		private static bool TryGetGroundAt(Vector3 xz, out Vector3 hitPoint, out Vector3 hitNormal)
		{
			//IL_0002: Unknown result type (might be due to invalid IL or missing references)
			//IL_0008: Unknown result type (might be due to invalid IL or missing references)
			//IL_0013: Unknown result type (might be due to invalid IL or missing references)
			//IL_0019: Unknown result type (might be due to invalid IL or missing references)
			//IL_001e: Unknown result type (might be due to invalid IL or missing references)
			//IL_005a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0061: Unknown result type (might be due to invalid IL or missing references)
			//IL_0066: Unknown result type (might be due to invalid IL or missing references)
			//IL_0040: Unknown result type (might be due to invalid IL or missing references)
			//IL_0045: Unknown result type (might be due to invalid IL or missing references)
			//IL_004d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0052: Unknown result type (might be due to invalid IL or missing references)
			RaycastHit val = default(RaycastHit);
			if (Physics.Raycast(new Vector3(xz.x, rayHeight, xz.z), Vector3.down, ref val, rayHeight * 2f, groundMask, (QueryTriggerInteraction)1))
			{
				hitPoint = ((RaycastHit)(ref val)).point;
				hitNormal = ((RaycastHit)(ref val)).normal;
				return true;
			}
			hitPoint = default(Vector3);
			hitNormal = Vector3.up;
			return false;
		}

		private static bool TryGetStatByName(StatManager statManager, string statName, out Stat stat)
		{
			stat = null;
			if ((Object)(object)statManager == (Object)null || string.IsNullOrWhiteSpace(statName))
			{
				return false;
			}
			MethodInfo method = ((object)statManager).GetType().GetMethod("GetStat", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, new Type[2]
			{
				typeof(string),
				typeof(Stat).MakeByRefType()
			}, null);
			if (method == null)
			{
				return false;
			}
			object[] array = new object[2] { statName, stat };
			object obj = method.Invoke(statManager, array);
			bool flag = default(bool);
			int num;
			if (obj is bool)
			{
				flag = (bool)obj;
				num = 1;
			}
			else
			{
				num = 0;
			}
			if (((uint)num & (flag ? 1u : 0u)) != 0)
			{
				object obj2 = array[1];
				Stat val = (Stat)((obj2 is Stat) ? obj2 : null);
				if (val != null)
				{
					stat = val;
					return true;
				}
			}
			bool flag2 = default(bool);
			int num2;
			if (obj is bool)
			{
				flag2 = (bool)obj;
				num2 = 1;
			}
			else
			{
				num2 = 0;
			}
			return (byte)((uint)num2 & (flag2 ? 1u : 0u)) != 0;
		}

		private static Transform GetStakeHand()
		{
			//IL_0035: Unknown result type (might be due to invalid IL or missing references)
			//IL_0040: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			Controller val = (useRightHandForStakes ? (((Object)(object)current != (Object)null) ? current.RightController : null) : (((Object)(object)current != (Object)null) ? current.LeftController : null));
			if ((Object)val == (Object)null)
			{
				return null;
			}
			string value = (useRightHandForStakes ? "Right Hand" : "Left Hand");
			Transform[] componentsInChildren = ((Component)val).GetComponentsInChildren<Transform>(true);
			foreach (Transform val2 in componentsInChildren)
			{
				if (((Object)val2).name.EndsWith(value))
				{
					return val2;
				}
			}
			return ((Component)val).transform;
		}

		private static void TeleportPlayerTo(Vector3 worldPoint, Vector3 normal)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0014: Unknown result type (might be due to invalid IL or missing references)
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_001a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0036: Unknown result type (might be due to invalid IL or missing references)
			//IL_0037: Unknown result type (might be due to invalid IL or missing references)
			//IL_003d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0042: Unknown result type (might be due to invalid IL or missing references)
			//IL_0047: Unknown result type (might be due to invalid IL or missing references)
			//IL_004d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0052: Unknown result type (might be due to invalid IL or missing references)
			//IL_0057: Unknown result type (might be due to invalid IL or missing references)
			//IL_0060: Unknown result type (might be due to invalid IL or missing references)
			//IL_006b: Expected O, but got Unknown
			//IL_0029: Unknown result type (might be due to invalid IL or missing references)
			//IL_002e: Unknown result type (might be due to invalid IL or missing references)
			//IL_007c: Unknown result type (might be due to invalid IL or missing references)
			//IL_006e: Unknown result type (might be due to invalid IL or missing references)
			PlayerController current = PlayerController.Current;
			if (!((Object)current == (Object)null))
			{
				Vector3 val = -normal;
				if (((Vector3)(ref val)).sqrMagnitude < 1E-06f)
				{
					val = Vector3.forward;
				}
				((Vector3)(ref val)).Normalize();
				Vector3 val2 = worldPoint + normal * 0.05f + val * playerBackstep;
				PlayerLocomotionController component = ((Component)current).GetComponent<PlayerLocomotionController>();
				if ((Object)component != (Object)null)
				{
					((LocomotionController)component).MoveTo(val2, (LocomotionFunction)0);
				}
				else
				{
					((Component)current).transform.position = val2;
				}
			}
		}

		private static void PoseHandForStake(Transform hand, Vector3 groundPoint, Vector3 normal, Transform cam)
		{
			//IL_0001: Unknown result type (might be due to invalid IL or missing references)
			//IL_000c: Expected O, but got Unknown
			//IL_0010: Unknown result type (might be due to invalid IL or missing references)
			//IL_0011: Unknown result type (might be due to invalid IL or missing references)
			//IL_0017: Unknown result type (might be due to invalid IL or missing references)
			//IL_001c: Unknown result type (might be due to invalid IL or missing references)
			//IL_002d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0038: Expected O, but got Unknown
			//IL_0047: Unknown result type (might be due to invalid IL or missing references)
			//IL_003a: Unknown result type (might be due to invalid IL or missing references)
			//IL_004c: Unknown result type (might be due to invalid IL or missing references)
			//IL_004d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0052: Unknown result type (might be due to invalid IL or missing references)
			//IL_008e: Unknown result type (might be due to invalid IL or missing references)
			//IL_008f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0090: Unknown result type (might be due to invalid IL or missing references)
			//IL_0095: Unknown result type (might be due to invalid IL or missing references)
			//IL_0096: Unknown result type (might be due to invalid IL or missing references)
			//IL_0097: Unknown result type (might be due to invalid IL or missing references)
			//IL_009c: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
			//IL_00af: Unknown result type (might be due to invalid IL or missing references)
			//IL_00b4: Unknown result type (might be due to invalid IL or missing references)
			//IL_00b6: Unknown result type (might be due to invalid IL or missing references)
			//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
			//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
			//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
			//IL_0061: Unknown result type (might be due to invalid IL or missing references)
			//IL_0062: Unknown result type (might be due to invalid IL or missing references)
			//IL_0067: Unknown result type (might be due to invalid IL or missing references)
			//IL_006c: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ed: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
			//IL_007b: Unknown result type (might be due to invalid IL or missing references)
			//IL_007c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0081: Unknown result type (might be due to invalid IL or missing references)
			//IL_0086: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ea: Unknown result type (might be due to invalid IL or missing references)
			if ((Object)hand == (Object)null)
			{
				return;
			}
			hand.position = groundPoint + normal * handHeightAboveGround;
			PlayerController current = PlayerController.Current;
			Vector3 val = Vector3.ProjectOnPlane(((Object)current != (Object)null) ? ((Component)current).transform.forward : Vector3.forward, normal);
			if (((Vector3)(ref val)).sqrMagnitude < 1E-06f)
			{
				val = Vector3.Cross(normal, Vector3.right);
				if (((Vector3)(ref val)).sqrMagnitude < 1E-06f)
				{
					val = Vector3.Cross(normal, Vector3.forward);
				}
			}
			((Vector3)(ref val)).Normalize();
			Quaternion val2 = Quaternion.LookRotation(val, normal);
			Vector3 val3 = val2 * Vector3.right;
			Quaternion val4 = Quaternion.AngleAxis(-90f, val3);
			Quaternion val5 = Quaternion.identity;
			Vector3 val6 = Vector3.Cross(normal, val);
			if (handTiltDegrees != 0f && ((Vector3)(ref val6)).sqrMagnitude > 1E-06f)
			{
				val5 = Quaternion.AngleAxis(handTiltDegrees, ((Vector3)(ref val6)).normalized);
			}
			hand.rotation = val5 * val4 * val2;
		}

		public static IEnumerator OutlineChunkWithStakes()
		{
			outlineRoutineRunning = true;
			PlayerController current = PlayerController.Current;
			if ((Object)current == (Object)null)
			{
				outlineRoutineRunning = false;
				yield break;
			}
			Transform hand = GetStakeHand();
			if ((Object)hand == (Object)null)
			{
				MelonLogger.Warning("[OutlineChunks] Could not find stake hand.");
				outlineRoutineRunning = false;
				yield break;
			}
			Transform cam = (((Object)current.Camera != (Object)null) ? ((Component)current.Camera).transform : null);
			foreach (Vector3 item in GeneratePerimeterTargets(((Component)current).transform.position))
			{
				if (TryGetGroundAt(item, out var ground, out var normal))
				{
					TeleportPlayerTo(ground, normal);
					yield return null;
					PoseHandForStake(hand, ground, normal, cam);
					GrabStake();
					yield return (object)new WaitForSeconds(0.05f);
					DropStakeAt(ground, normal);
					yield return (object)new WaitForSeconds(0.05f);
				}
				ground = default(Vector3);
				normal = default(Vector3);
			}
			MelonLogger.Msg("[OutlineChunks] Finished placing stakes around current chunk.");
			outlineRoutineRunning = false;
		}

		private static void GrabStake()
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Expected O, but got Unknown
			//IL_002a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0035: Expected O, but got Unknown
			//IL_00df: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ec: Expected O, but got Unknown
			//IL_0038: Unknown result type (might be due to invalid IL or missing references)
			//IL_0043: Expected O, but got Unknown
			//IL_0092: Unknown result type (might be due to invalid IL or missing references)
			//IL_0098: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
			//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
			if ((Object)PlayerController.Current == (Object)null)
			{
				MelonLogger.Warning("[DrawMode] No player.");
				return;
			}
			Transform stakeHand = GetStakeHand();
			Interactor stakeInteractor = GetStakeInteractor();
			if ((Object)stakeHand == (Object)null || (Object)stakeInteractor == (Object)null)
			{
				MelonLogger.Warning("[DrawMode] Missing stake hand or interactor.");
				return;
			}
			if (stakeInteractor.IsInteracting)
			{
				stakeInteractor.StopInteract(false, false);
			}
			if (!TryFindStakePickup(out var pu, out var tr, out var _))
			{
				MelonLogger.Warning("[DrawMode] Could not locate a Wooden Stake under player or in scene.");
				return;
			}
			if (!((Component)tr).gameObject.activeInHierarchy)
			{
				((Component)tr).gameObject.SetActive(true);
			}
			if (Vector3.Distance(tr.position, stakeHand.position) > 0.4f)
			{
				tr.position = stakeHand.position;
				tr.rotation = stakeHand.rotation;
			}
			if (pu.IsDocked)
			{
				try
				{
					pu.Undock(stakeInteractor, true, 1);
				}
				catch
				{
				}
			}
			stakeInteractor.ResetTimeout();
			stakeInteractor.StartInteract((Interactable)pu, false, false, false);
			_heldStake = pu;
		}

		private static void DropStakeAt(Vector3 ground, Vector3 normal)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0029: Unknown result type (might be due to invalid IL or missing references)
			//IL_0034: Expected O, but got Unknown
			//IL_0043: Unknown result type (might be due to invalid IL or missing references)
			//IL_0048: Unknown result type (might be due to invalid IL or missing references)
			//IL_0049: Unknown result type (might be due to invalid IL or missing references)
			//IL_004f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0054: Unknown result type (might be due to invalid IL or missing references)
			//IL_005f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0060: Unknown result type (might be due to invalid IL or missing references)
			//IL_0066: Unknown result type (might be due to invalid IL or missing references)
			//IL_006b: Unknown result type (might be due to invalid IL or missing references)
			Interactor stakeInteractor = GetStakeInteractor();
			if (!((Object)stakeInteractor == (Object)null))
			{
				if (stakeInteractor.IsInteracting)
				{
					stakeInteractor.StopInteract(false, false);
				}
				if ((Object)_heldStake != (Object)null)
				{
					Transform transform = ((Component)_heldStake).transform;
					transform.rotation = Quaternion.FromToRotation(transform.up, normal) * transform.rotation;
					transform.position = ground + normal * 0.02f;
					_heldStake = null;
				}
			}
		}

		private static Transform FindStakeOnLocalPlayer()
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_001f: Unknown result type (might be due to invalid IL or missing references)
			//IL_002a: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			Transform val = (((Object)current != (Object)null) ? ((Component)current).transform : null);
			if ((Object)val == (Object)null)
			{
				return null;
			}
			Transform[] componentsInChildren = ((Component)val).GetComponentsInChildren<Transform>(true);
			foreach (Transform val2 in componentsInChildren)
			{
				if (((Object)val2).name.Contains("Wooden Stake(Clone)"))
				{
					return val2;
				}
			}
			return null;
		}

		private static bool TryFindStakePickup(out Pickup pu, out Transform tr, out bool fromScene)
		{
			//IL_0023: Unknown result type (might be due to invalid IL or missing references)
			//IL_002e: Expected O, but got Unknown
			//IL_00b4: Unknown result type (might be due to invalid IL or missing references)
			//IL_00bf: Expected O, but got Unknown
			//IL_00c8: Unknown result type (might be due to invalid IL or missing references)
			//IL_00d3: Expected O, but got Unknown
			//IL_00d7: Unknown result type (might be due to invalid IL or missing references)
			pu = null;
			tr = null;
			fromScene = false;
			PlayerController current = PlayerController.Current;
			Transform val = (((Object)(object)current != (Object)null) ? ((Component)current).transform : null);
			if ((Object)val != (Object)null)
			{
				Pickup[] componentsInChildren = ((Component)val).GetComponentsInChildren<Pickup>(true);
				foreach (Pickup val2 in componentsInChildren)
				{
					string text = ((Object)val2).name ?? ((Object)((Component)val2).gameObject).name ?? string.Empty;
					if (text.IndexOf("Wooden Stake", StringComparison.OrdinalIgnoreCase) >= 0 || text.IndexOf("Stake", StringComparison.OrdinalIgnoreCase) >= 0)
					{
						pu = val2;
						tr = ((Component)val2).transform;
						fromScene = false;
						return true;
					}
				}
			}
			try
			{
				Pickup[] componentsInChildren = Resources.FindObjectsOfTypeAll<Pickup>();
				foreach (Pickup val3 in componentsInChildren)
				{
					if (!((Object)val3 == (Object)null) && !((Object)((Component)val3).gameObject == (Object)null) && (int)((Object)val3).hideFlags == 0)
					{
						string text2 = ((Object)val3).name ?? ((Object)((Component)val3).gameObject).name ?? string.Empty;
						if (text2.IndexOf("Wooden Stake", StringComparison.OrdinalIgnoreCase) >= 0 || text2.IndexOf("Stake", StringComparison.OrdinalIgnoreCase) >= 0)
						{
							pu = val3;
							tr = ((Component)val3).transform;
							fromScene = true;
							return true;
						}
					}
				}
			}
			catch
			{
			}
			return false;
		}

		private static Interactor GetStakeInteractor()
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0043: Unknown result type (might be due to invalid IL or missing references)
			//IL_004e: Expected O, but got Unknown
			//IL_0025: Unknown result type (might be due to invalid IL or missing references)
			//IL_0030: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			if ((Object)current == (Object)null)
			{
				return null;
			}
			if (!useRightHandForStakes)
			{
				Controller leftController = current.LeftController;
				if ((Object)leftController == (Object)null)
				{
					return null;
				}
				return leftController.Interactor;
			}
			Controller rightController = current.RightController;
			if ((Object)rightController == (Object)null)
			{
				return null;
			}
			return rightController.Interactor;
		}

		private static IEnumerable<Vector3> GeneratePerimeterTargets(Vector3 around)
		{
			//IL_0008: Unknown result type (might be due to invalid IL or missing references)
			//IL_0009: Unknown result type (might be due to invalid IL or missing references)
			float r = 8f;
			int steps = 16;
			for (int i = 0; i < steps; i++)
			{
				float num = (float)i / (float)steps * (float)Math.PI * 2f;
				yield return around + new Vector3(Mathf.Cos(num) * r, 0f, Mathf.Sin(num) * r);
			}
		}
	}

	private enum MenuTab
	{
		Connect,
		General,
		Utilities
	}

	[Flags]
	private enum GripFlags
	{
		None = 0,
		Left = 1,
		Right = 2
	}

	private class PartnerHoldState
	{
		public bool Active;

		public bool IsRight;

		public Transform Hand;

		public Transform MySlot;

		public float BodyYOffset;

		public bool HasLastHandPos;

		public Vector3 LastHandPos;

		public Vector3 HandVelocity;
	}

	private class PartnerOrbitState
	{
		public bool Active;

		public Transform CenterHand;

		public Transform ControlHand;

		public float Radius;
	}

	private class MarkerInfo
	{
		public Transform target;

		public GameObject ball;

		public GameObject labelRoot;

		public TextMesh labelTM;

		public Vector3 baseScale;

		public float phaseJitter;

		public bool parented;
	}

	internal static class Actions
	{
		public struct PlayerEffectTeleportInformation
		{
			public Vector3 teleportEffectPosition;

			public TeleportType teleportType;
		}

		[HarmonyPatch(typeof(VivoxVoice), "Alta.Voice.IVivoxVoiceUpdatable.Update3DPosition")]
		public static class Patch_VivoxVoice_Update3DPosition
		{
			private static bool Prefix(VivoxVoice __instance)
			{
				//IL_0031: Unknown result type (might be due to invalid IL or missing references)
				//IL_0036: Unknown result type (might be due to invalid IL or missing references)
				//IL_0038: Unknown result type (might be due to invalid IL or missing references)
				//IL_003d: Unknown result type (might be due to invalid IL or missing references)
				//IL_003e: Unknown result type (might be due to invalid IL or missing references)
				//IL_0043: Unknown result type (might be due to invalid IL or missing references)
				//IL_0072: Unknown result type (might be due to invalid IL or missing references)
				//IL_0079: Expected O, but got Unknown
				//IL_007f: Unknown result type (might be due to invalid IL or missing references)
				//IL_0080: Unknown result type (might be due to invalid IL or missing references)
				//IL_0081: Unknown result type (might be due to invalid IL or missing references)
				//IL_0082: Unknown result type (might be due to invalid IL or missing references)
				if (!Basemodal.main_values.is_orb_movement)
				{
					return true;
				}
				float realtimeSinceStartup = Time.realtimeSinceStartup;
				if (realtimeSinceStartup > nextTime)
				{
					nextTime = realtimeSinceStartup + 0.3f;
					Transform orb_model = get_orb().orb_model;
					Vector3 position = orb_model.position;
					Vector3 up = orb_model.up;
					Vector3 forward = orb_model.forward;
					object value = AccessTools.Field(typeof(VivoxVoice), "session").GetValue(__instance);
					if (value != null)
					{
						IChannelSession val = (IChannelSession)((value is IChannelSession) ? value : null);
						if (val != null)
						{
							val.Set3DPosition(position, position, forward, up);
							return false;
						}
					}
					MelonLogger.Msg($"session could not be found or is not a IChannelSession. type: {value?.GetType()}");
				}
				return false;
			}
		}

		[HarmonyPatch(typeof(PlayerDocks), "Filter")]
		public static class PlayerDocks_Filter_Patch
		{
			public static bool Prefix(PlayerDocks __instance)
			{
				AccessTools.Field(typeof(PlayerDocks), "owner")?.SetValue(__instance, PlayerController.Current);
				return true;
			}
		}

		[HarmonyPatch(typeof(TradeVendor), "Initialize")]
		public static class trade_deck_Patch
		{
			public static void Postfix(TradeVendor __instance)
			{
				do_smth(__instance);
			}

			private static async Task do_smth(TradeVendor instance)
			{
				await Task.Delay(10000);
				MelonLogger.Msg("ejected thing");
				AccessTools.Method(((object)instance).GetType(), "SetOwnerPlayer", (Type[])null, (Type[])null)?.Invoke(instance, new object[1] { PlayerController.Current.Info.Player });
			}
		}

		[HarmonyPatch(typeof(QuickAccessMenuAction), "InitializeForPlayer")]
		public static class InitializeForPlayer_Patch
		{
			public static bool Prefix(QuickAccessMenuAction __instance)
			{
				//IL_000c: Unknown result type (might be due to invalid IL or missing references)
				//IL_0012: Expected O, but got Unknown
				ShowNames val = (ShowNames)((__instance is ShowNames) ? __instance : null);
				if ((Object)(object)val != (Object)null)
				{
					MelonLogger.Msg("ShowNames InitializeForPlayer triggered");
					shownames = val;
					((QuickAccessMenuAction)val).Run(PlayerController.Current.RightController);
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(MapBoard), "SetupMapPieces")]
		public static class MapBoard_Patch
		{
			public static void Prefix(MapBoard __instance, PlayerMapManager mapManager)
			{
				for (int i = 0; i < 17; i++)
				{
					mapManager.DiscoverPiece(__instance.Map, i);
				}
			}
		}

		[HarmonyPatch(typeof(PlayerCommunicationManager), "HandlePlayerMessage")]
		public static class PlayerCommunicationManager_Patch
		{
			public static bool Prefix(IPlayer player, Stream stream, PlayerTextMessage textMessage)
			{
				((PlayerTextMessage)(ref textMessage)).Serialize(player, stream);
				if (stream.IsWriting)
				{
					return false;
				}
				if (string.IsNullOrWhiteSpace(((PlayerTextMessage)(ref textMessage)).Message))
				{
					return false;
				}
				MelonLogger.Msg("message received: " + ((PlayerTextMessage)(ref textMessage)).Message);
				return false;
			}
		}

		[HarmonyPatch(typeof(PickupDock))]
		public static class PickupDock_CheckForward_Patch
		{
			[HarmonyPatch("CheckForward")]
			[HarmonyPrefix]
			public static bool Prefix_CheckForward(PickupDock __instance, Interactor interactor, ref bool isForced)
			{
				AccessTools.PropertySetter(((object)__instance).GetType(), "PlayerController")?.Invoke(__instance, new object[1] { PlayerController.Current });
				return true;
			}
		}

		[HarmonyPatch]
		public static class PickupDock_CheckAllowed_Patch
		{
			public static MethodBase TargetMethod()
			{
				return typeof(PickupDock).GetMethod("CheckAllowed", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, new Type[4]
				{
					typeof(Interactor),
					typeof(Pickup),
					typeof(int),
					typeof(PickupDock).MakeByRefType()
				}, null);
			}

			[HarmonyPrefix]
			public static bool Prefix(PickupDock __instance, Interactor interactor, Pickup pickup, int quantity, ref PickupDock finalDock)
			{
				//IL_0006: Unknown result type (might be due to invalid IL or missing references)
				//IL_0010: Unknown result type (might be due to invalid IL or missing references)
				//IL_001a: Expected O, but got Unknown
				//IL_001a: Expected O, but got Unknown
				if ((Object)__instance.PlayerController != (Object)PlayerController.Current)
				{
					AccessTools.PropertySetter(((object)__instance).GetType(), "PlayerController")?.Invoke(__instance, new object[1] { PlayerController.Current });
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(ImpactDetector), "OnToolHit")]
		public static class ImpactDetector_patch
		{
			public static bool Prefix(ImpactDetector __instance, ImpactHitWithTool impactWithTool)
			{
				//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
				if (impactWithTool.WasByPlayer)
				{
					Player hittingPlayer = impactWithTool.HittingPlayer;
					int? num;
					if ((Object)(object)hittingPlayer == (Object)null)
					{
						num = null;
					}
					else
					{
						UserInfoAndRole userInfo = hittingPlayer.UserInfo;
						num = ((userInfo != null) ? new int?(userInfo.Identifier) : ((int?)null));
					}
					int? num2 = num;
					IPlayer current = Player.Current;
					int? num3;
					if (current == null)
					{
						num3 = null;
					}
					else
					{
						UserInfoAndRole userInfo2 = current.UserInfo;
						num3 = ((userInfo2 != null) ? new int?(userInfo2.Identifier) : ((int?)null));
					}
					if (num2 == num3)
					{
						_lastImpactHitWithTool = impactWithTool;
						ImpactList.Add((impactWithTool, __instance, Vector3.zero));
					}
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(SmoothLocomotion), "MoveToSafety")]
		public static class no_void_tp_patch
		{
			public static bool Prefix()
			{
				return !no_void_tp;
			}
		}

		private static FriendlyOrbBehavior2 _controlledOrb;

		public static readonly List<(ImpactHitWithTool Impact, ImpactDetector Detector, Vector3 HitPoint)> ImpactList = new List<(ImpactHitWithTool, ImpactDetector, Vector3)>();

		private static ImpactHitWithTool _lastImpactHitWithTool;

		public static bool no_void_tp = true;

		public static float YOffset = -0.5f;

		public static bool is_devlight;

		private static OpenXRControllerInput leftInput;

		private static OpenXRControllerInput rightInput;

		private static Transform leftHand;

		private static Transform rightHand;

		private static Transform headCtrl;

		private static ShowNames showNames;

		public static Transform master_terrain;

		public static Vector3 base_terrain_pos;

		private static OpenXRControllerInput leftcontrollerInput;

		private static OpenXRControllerInput rightcontrollerInput;

		private static Transform left_hand;

		private static Transform right_hand;

		private static Transform head_controller;

		private static ShowNames shownames;

		private static FriendlyOrbBehavior2 controlled_orb;

		private static float nextTime;

		private static MethodSyncStruct<PlayerEffectTeleportInformation> remoteTeleportEffect;

		public static FriendlyOrbBehavior2 get_orb()
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Expected O, but got Unknown
			//IL_007f: Unknown result type (might be due to invalid IL or missing references)
			//IL_008a: Expected O, but got Unknown
			if ((Object)_controlledOrb != (Object)null)
			{
				return _controlledOrb;
			}
			IPlayer current = Player.Current;
			int? num;
			if (current == null)
			{
				num = null;
			}
			else
			{
				UserInfoAndRole userInfo = current.UserInfo;
				num = ((userInfo != null) ? new int?(userInfo.Identifier) : ((int?)null));
			}
			int num2 = num ?? 0;
			int num3 = num2;
			if (num3 != 0)
			{
				_controlledOrb = OrbUtils.get_orb(num3);
			}
			if ((Object)_controlledOrb == (Object)null)
			{
				_controlledOrb = Object.FindObjectOfType<FriendlyOrbBehavior2>();
			}
			return _controlledOrb;
		}

		public static void take_control()
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			FriendlyOrbBehavior2 orb = get_orb();
			if ((Object)orb != (Object)null && !orb.is_being_controlled)
			{
				orb.take_control(local: true);
			}
		}

		public static void exit_control()
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			FriendlyOrbBehavior2 orb = get_orb();
			if ((Object)orb == (Object)null)
			{
				_controlledOrb = null;
				return;
			}
			if (orb.is_being_controlled)
			{
				orb.exit_control(local: true);
			}
			_controlledOrb = null;
		}

		public static void ClearImpactCache()
		{
			ImpactList.Clear();
			_lastImpactHitWithTool = null;
		}

		public static void toggle_devlight()
		{
			//IL_005d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0063: Expected O, but got Unknown
			is_devlight = !is_devlight;
			PlayerController.Current.PlayerLight.SetPlayerLightEnabled(is_devlight);
			object obj = AccessTools.Field(((object)PlayerController.Current.PlayerLight).GetType(), "platformLight")?.GetValue(PlayerController.Current.PlayerLight);
			PlatformLight val = (PlatformLight)((obj is PlatformLight) ? obj : null);
			if ((Object)(object)val != (Object)null)
			{
				val.IsActive = is_devlight;
				((Behaviour)val).enabled = is_devlight;
				val.Range = 100f;
			}
		}

		public static void set_speed(float speed)
		{
			//IL_0012: Unknown result type (might be due to invalid IL or missing references)
			//IL_0018: Expected O, but got Unknown
			//IL_001b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0026: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			PlayerCharacter val = (PlayerCharacter)((current is PlayerCharacter) ? current : null);
			Stat stat = null;
			if (!((Object)val == (Object)null) && TryGetStatByName(((HealthObject)val.Health).StatManager, "speed", out stat))
			{
				stat.Base = speed;
			}
		}

		public static void escape_customization()
		{
			TeleporterPlatform[] array = Object.FindObjectsOfType<TeleporterPlatform>();
			foreach (TeleporterPlatform val in array)
			{
				if (((Object)((Component)val).transform).name == "Customization_Teleporter")
				{
					FieldInfo fieldInfo = AccessTools.Field(((object)val).GetType(), "isLocalPressing");
					if (fieldInfo != null)
					{
						fieldInfo.SetValue(val, true);
					}
					FieldInfo fieldInfo2 = AccessTools.Field(((object)val).GetType(), "localEndTime");
					if (fieldInfo2 != null)
					{
						fieldInfo2.SetValue(val, -100f);
					}
					AccessTools.Method(typeof(TeleporterPlatform), "UpdateProgress", (Type[])null, (Type[])null)?.Invoke(val, Array.Empty<object>());
				}
			}
		}

		public static void toggle_mic()
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			VivoxManager instance = SingletonBehaviour<VivoxManager>.Instance;
			if ((Object)instance != (Object)null)
			{
				instance.Client.ToggleMute();
				PlayerController.Current.MessageDisplay.Display(instance.Client.IsMicMuted ? "muted mic" : "unmuted mic", 1f, (DisplayMessageType)2);
			}
		}

		public static void toggle_names()
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Expected O, but got Unknown
			//IL_002c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0033: Expected O, but got Unknown
			if ((Object)showNames == (Object)null)
			{
				QuickAccessMenuAction[] array = Resources.FindObjectsOfTypeAll<QuickAccessMenuAction>();
				foreach (QuickAccessMenuAction val in array)
				{
					ShowNames val2 = (ShowNames)((val is ShowNames) ? val : null);
					if ((Object)(object)val2 != (Object)null)
					{
						showNames = val2;
						break;
					}
				}
			}
			ShowNames val3 = showNames;
			if ((Object)(object)val3 != (Object)null)
			{
				((QuickAccessMenuAction)val3).Run(PlayerController.Current.RightController);
			}
		}

		public static void get_player_list()
		{
			string arg = new string('-', 20);
			string text = $"Online Players ({Player.AllPlayers.Count})\n{arg}\n";
			foreach (IPlayer allPlayer in Player.AllPlayers)
			{
				text = text + allPlayer.UserInfo.Username + "\n";
			}
			PlayerController.Current.MessageDisplay.Display(text, 5f, (DisplayMessageType)2);
		}

		public static void change_fov(float fov)
		{
			//IL_001a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0025: Expected O, but got Unknown
			Transform val = find_head();
			Camera val2 = (((Object)(object)val != (Object)null) ? ((Component)val).GetComponent<Camera>() : null);
			if ((Object)val2 != (Object)null)
			{
				val2.fieldOfView = fov;
			}
		}

		public static Transform find_head()
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Expected O, but got Unknown
			//IL_003f: Unknown result type (might be due to invalid IL or missing references)
			//IL_004a: Expected O, but got Unknown
			//IL_0058: Unknown result type (might be due to invalid IL or missing references)
			//IL_0063: Expected O, but got Unknown
			//IL_0099: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a4: Expected O, but got Unknown
			//IL_0077: Unknown result type (might be due to invalid IL or missing references)
			//IL_0082: Expected O, but got Unknown
			if ((Object)headCtrl != (Object)null && (Object)(object)headCtrl != (Object)null)
			{
				return headCtrl;
			}
			PlayerController current = PlayerController.Current;
			Transform val = (((Object)(object)current != (Object)null) ? current.Head : null);
			if ((Object)val == (Object)null)
			{
				return null;
			}
			Transform parent = val.parent;
			Transform val2 = null;
			if ((Object)parent != (Object)null)
			{
				val2 = parent.Find("Height Fixer") ?? parent;
				val2 = (((Object)val2 != (Object)null) ? (val2.Find("VR Head (eye)") ?? val2) : null);
			}
			headCtrl = (((Object)val2 != (Object)null) ? val2 : val);
			return headCtrl;
		}

		public static Transform find_hand(bool right)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_007d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0088: Expected O, but got Unknown
			//IL_001e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0029: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			if ((Object)current == (Object)null)
			{
				return null;
			}
			if (right)
			{
				if ((Object)rightHand != (Object)null && (Object)(object)rightHand != (Object)null)
				{
					return rightHand;
				}
				Controller rightController = current.RightController;
				Hand val = (((Object)(object)rightController != (Object)null) ? rightController.Hand : null);
				rightHand = (((Object)(object)val != (Object)null) ? ((Component)val).transform : null);
				return rightHand;
			}
			if ((Object)leftHand != (Object)null && (Object)(object)leftHand != (Object)null)
			{
				return leftHand;
			}
			Controller leftController = current.LeftController;
			Hand val2 = (((Object)(object)leftController != (Object)null) ? leftController.Hand : null);
			leftHand = (((Object)(object)val2 != (Object)null) ? ((Component)val2).transform : null);
			return leftHand;
		}

		public static OpenXRControllerInput find_controller(bool right)
		{
			//IL_0069: Unknown result type (might be due to invalid IL or missing references)
			//IL_006f: Expected O, but got Unknown
			DeviceInputController val = (right ? PlayerController.Current.Info.RightController.PlayerInput.InputController : PlayerController.Current.Info.LeftController.PlayerInput.InputController);
			PropertyInfo propertyInfo = AccessTools.Property(((object)val).GetType(), "Input");
			object obj = ((propertyInfo != null) ? propertyInfo.GetValue(val) : null);
			OpenXRControllerInput val2 = (OpenXRControllerInput)((obj is OpenXRControllerInput) ? obj : null);
			if (val2 == null)
			{
				return null;
			}
			if (right)
			{
				rightInput = val2;
			}
			else
			{
				leftInput = val2;
			}
			return val2;
		}

		public static async Task revive()
		{
			Transform head = find_head();
			if ((Object)head == (Object)null)
			{
				return;
			}
			move_world_hand("right", head.position);
			while (true)
			{
				Vector3 val = find_hand(right: true).position - head.position;
				if (!(((Vector3)(ref val)).magnitude < 0.02f))
				{
					val = find_hand(right: false).position - head.position;
					if (!(((Vector3)(ref val)).magnitude < 0.02f))
					{
						break;
					}
				}
				move_world_hand("right", head.position);
				await Task.Delay(100);
			}
			ReviveOrb[] array = Object.FindObjectsOfType<ReviveOrb>();
			for (int i = 0; i < array.Length; i++)
			{
				Pickup component = ((Component)((Component)array[i]).transform).GetComponent<Pickup>();
				((Behaviour)component).enabled = true;
				((Interactable)component).Interact(PlayerController.Current.RightController.Interactor);
				PlayerController.Current.RightController.Interactor.StartInteract((Interactable)component, false, false, false);
				PlayerController.Current.RightController.Interactor.StopInteract(false, false);
				grab_item(right: false);
				drop_item(right: false);
			}
		}

		public static void grab_item(bool right)
		{
			OpenXRControllerInput val = find_controller(right);
			if (val != null)
			{
				((InputSource)((ControllerInput)val).Grab).State = true;
			}
		}

		public static void drop_item(bool right)
		{
			OpenXRControllerInput val = find_controller(right);
			if (val != null)
			{
				((InputSource)((ControllerInput)val).Grab).State = false;
			}
		}

		public static void use_smooth_locomotion(Vector2 direction)
		{
			//IL_000a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Invalid comparison between Unknown and I4
			//IL_001c: Unknown result type (might be due to invalid IL or missing references)
			OpenXRControllerInput val = find_controller((int)SettingsManager.GameConfig.Settings.SmoothLocomotionActiveHand > 0);
			if (val != null)
			{
				((ControllerInput)val).SmoothLocomotion = direction;
			}
		}

		public static void move_head(Vector3 local_position, Vector3? rotation = null)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_0027: Unknown result type (might be due to invalid IL or missing references)
			//IL_002c: Unknown result type (might be due to invalid IL or missing references)
			Transform val = find_head();
			if (!((Object)val == (Object)null))
			{
				val.localPosition = local_position;
				if (rotation.HasValue)
				{
					val.localRotation = Quaternion.Euler(rotation.Value);
				}
			}
		}

		public static void move_head_world(Vector3 world_position, Vector3? rotation = null)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_0027: Unknown result type (might be due to invalid IL or missing references)
			//IL_002c: Unknown result type (might be due to invalid IL or missing references)
			Transform val = find_head();
			if (!((Object)val == (Object)null))
			{
				val.position = world_position;
				if (rotation.HasValue)
				{
					val.rotation = Quaternion.Euler(rotation.Value);
				}
			}
		}

		public static void move_world_hand(string hand, Vector3 pos, Vector3? rotation = null)
		{
			//IL_006d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0078: Unknown result type (might be due to invalid IL or missing references)
			//IL_0079: Unknown result type (might be due to invalid IL or missing references)
			//IL_007e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0083: Unknown result type (might be due to invalid IL or missing references)
			//IL_0094: Unknown result type (might be due to invalid IL or missing references)
			//IL_0095: Unknown result type (might be due to invalid IL or missing references)
			//IL_009a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0052: Unknown result type (might be due to invalid IL or missing references)
			//IL_0057: Unknown result type (might be due to invalid IL or missing references)
			//IL_005c: Unknown result type (might be due to invalid IL or missing references)
			OpenXRControllerInput val = find_controller(hand == "right");
			if (val != null)
			{
				pos.y += YOffset;
				Transform transform = ((Component)PlayerController.Current.Info.RightController.Hand).transform;
				if (rotation.HasValue)
				{
					val.pose.rotation = Quaternion.Euler(rotation.Value);
				}
				val.pose.velocity = val.pose.position - transform.parent.InverseTransformPoint(pos);
				val.pose.position = transform.parent.InverseTransformPoint(pos);
			}
		}

		public static void move_hands(string hand, Vector3 pos, Vector3? rotation = null)
		{
			//IL_0035: Unknown result type (might be due to invalid IL or missing references)
			//IL_0036: Unknown result type (might be due to invalid IL or missing references)
			//IL_0067: Unknown result type (might be due to invalid IL or missing references)
			//IL_0068: Unknown result type (might be due to invalid IL or missing references)
			//IL_0052: Unknown result type (might be due to invalid IL or missing references)
			//IL_0057: Unknown result type (might be due to invalid IL or missing references)
			//IL_005c: Unknown result type (might be due to invalid IL or missing references)
			OpenXRControllerInput val = find_controller(hand == "right");
			if (val != null)
			{
				Transform transform = ((Component)PlayerController.Current.Info.RightController.Hand).transform;
				move_world_hand(hand, transform.parent.TransformPoint(pos), rotation);
				if (rotation.HasValue)
				{
					val.pose.rotation = Quaternion.Euler(rotation.Value);
				}
				val.pose.position = pos;
			}
		}

		public static void toggle_flight()
		{
			//IL_0012: Unknown result type (might be due to invalid IL or missing references)
			//IL_0018: Expected O, but got Unknown
			PlayerController current = PlayerController.Current;
			SmoothLocomotionPlayerController val = (SmoothLocomotionPlayerController)((current is SmoothLocomotionPlayerController) ? current : null);
			if ((Object)(object)val != (Object)null)
			{
				val.SmoothLocomotion.ToggleDebug();
			}
		}

		public static void AutoHarvestTick()
		{
			//IL_000a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0015: Expected O, but got Unknown
			//IL_0021: Unknown result type (might be due to invalid IL or missing references)
			//IL_002c: Expected O, but got Unknown
			//IL_0036: Unknown result type (might be due to invalid IL or missing references)
			//IL_0041: Expected O, but got Unknown
			//IL_0045: Unknown result type (might be due to invalid IL or missing references)
			//IL_004b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0055: Unknown result type (might be due to invalid IL or missing references)
			//IL_005a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0061: Unknown result type (might be due to invalid IL or missing references)
			//IL_0068: Unknown result type (might be due to invalid IL or missing references)
			//IL_006d: Unknown result type (might be due to invalid IL or missing references)
			//IL_008e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0099: Expected O, but got Unknown
			if ((Object)_lastImpactHitWithTool.Tool == (Object)null || (Object)_lastImpactHitWithTool.Impactor == (Object)null)
			{
				return;
			}
			Transform val = find_head();
			if ((Object)val == (Object)null)
			{
				return;
			}
			Vector3 val2 = val.position + val.forward * 0.2f;
			RaycastHit val3 = default(RaycastHit);
			if (Physics.Raycast(new Ray(val2, val.forward), ref val3, 50f))
			{
				ImpactDetector component = ((Component)((RaycastHit)(ref val3)).transform).GetComponent<ImpactDetector>();
				if (!((Object)component == (Object)null))
				{
					AccessTools.Field(((object)component).GetType(), "syncImpact")?.GetValue(component);
				}
			}
		}

		public static void toggle_terrain()
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0054: Unknown result type (might be due to invalid IL or missing references)
			//IL_0020: Unknown result type (might be due to invalid IL or missing references)
			if (master_terrain.position.y < base_terrain_pos.y)
			{
				master_terrain.position = base_terrain_pos;
			}
			else
			{
				master_terrain.position = new Vector3(base_terrain_pos.x, base_terrain_pos.y - 100f, base_terrain_pos.z);
			}
		}

		public static void player_message(string message, int duration = 3, Color? color = null, int font_size = 80)
		{
			//IL_002c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0031: Unknown result type (might be due to invalid IL or missing references)
			//IL_0038: Unknown result type (might be due to invalid IL or missing references)
			//IL_003b: Unknown result type (might be due to invalid IL or missing references)
			//IL_004b: Expected O, but got Unknown
			//IL_000b: Unknown result type (might be due to invalid IL or missing references)
			if (!color.HasValue)
			{
				color = Color.white;
			}
			PlayerController.Current.MessageDisplay.Display(new DisplayMessage(message, (float)duration, 0.1f, 0.1f)
			{
				FontSize = font_size,
				FontColor = color.Value
			}, (DisplayMessageType)2);
		}

		public static async Task rejoin_async(int attempts = 0)
		{
			if (!Basemodal.main_values.in_kicked_dimension)
			{
				return;
			}
			if (attempts > 10)
			{
				player_message("rejoin threshold has been reached. stopped trying to rejoin.", 3, (Color?)new Color(255f, 221f, 51f), 80);
				return;
			}
			ServerJoinResult res = await prejoin(Basemodal.main_values.target_server_id);
			if (res.IsAllowed)
			{
				Basemodal.main_values.join_info = res;
				player_message("successfully rejoined :>\nmake sure to leave the kicked dimension before pressing join", 3, Color.green);
				return;
			}
			if ((int)res.FailReason == 10)
			{
				player_message("you have been banned.\ntry to talk your way out of it or something", 3, Color.red);
				return;
			}
			if ((int)res.FailReason == 12)
			{
				player_message("a admin thinks they are better then you.\nretrying rejoin", 3, (Color?)new Color(255f, 221f, 51f), 80);
				attempts++;
				await rejoin_async(attempts);
			}
			if ((int)res.FailReason != 11 && (int)res.FailReason != 9 && (int)res.FailReason != 1 && (int)res.FailReason != 2)
			{
				player_message("failed to rejoin but retrying...\n" + res.Message, 3, (Color?)new Color(255f, 221f, 51f), 80);
				attempts++;
				await rejoin_async(attempts);
			}
		}

		public static async Task<ServerJoinResult> prejoin(int server_id)
		{
			if (!IsServerAllowed(server_id))
			{
				QuitForDisallowed("Prejoin id=" + server_id);
				return await BuildDeniedJoinResult("Server ID is not in the allowed list.", server_id);
			}
			GenericVersionParts val = VersionHelper.Parse(((object)BuildVersion.GetCurrentVersion()).ToString());
			GameVersion val2 = new GameVersion(val.Stream, val.Season, val.Major, val.Minor, val.ChangeSet);
			ServerJoinResult val3 = await ApiAccess.ApiClient.ServerClient.JoinServer(server_id, val2, "ima person", JwtTokenHandler.Write(ApiAccess.ApiClient.UserCredentials.IdentityToken), true, true);
			MelonLogger.Msg("running import api request");
			MelonLogger.Msg(val3.IsAllowed ? "join server was successful" : $"not allowed to join server: {val3.FailReason}");
			if (val3.IsAllowed)
			{
				MelonLogger.Msg($"join data:\nallowed: {val3.IsAllowed}\naddress: {val3.ConnectionInfo.Address}\nport:{val3.ConnectionInfo.GamePort}\nws port: {val3.ConnectionInfo.WebServerPort + 1}\ntoken: {JwtTokenHandler.Write(val3.Token)}");
			}
			return val3;
		}

		public static void use_orb_movement(Vector3 target_pos)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0023: Unknown result type (might be due to invalid IL or missing references)
			//IL_0024: Unknown result type (might be due to invalid IL or missing references)
			//IL_003e: Unknown result type (might be due to invalid IL or missing references)
			FriendlyOrbBehavior2 orb = get_orb();
			if (!((Object)orb == (Object)null))
			{
				if (!orb.is_being_controlled)
				{
					orb.take_control(local: true);
				}
				if (target_pos == Vector3.zero)
				{
					orb.TargetPosition = null;
				}
				else
				{
					orb.TargetPosition = target_pos;
				}
			}
		}

		public static void rotate_orb(Quaternion target_rot)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			//IL_0024: Unknown result type (might be due to invalid IL or missing references)
			//IL_0025: Unknown result type (might be due to invalid IL or missing references)
			FriendlyOrbBehavior2 orb = get_orb();
			if (!((Object)orb == (Object)null))
			{
				if (!orb.is_being_controlled)
				{
					orb.take_control(local: true);
				}
				orb.target_rotation = target_rot;
			}
		}

		public static Player get_player(string username)
		{
			//IL_0034: Unknown result type (might be due to invalid IL or missing references)
			//IL_003a: Expected O, but got Unknown
			foreach (IPlayer allPlayer in Player.AllPlayers)
			{
				if (allPlayer.UserInfo.Username == username)
				{
					return (Player)((allPlayer is Player) ? allPlayer : null);
				}
			}
			MelonLogger.Msg("failed to find " + username + " in the online players");
			return null;
		}

		private static async Task update_nameboard(NameBoard __instance, bool isFriend, string title, string userName, PooledObjectDefinition shieldDefinition)
		{
			MelonLogger.Msg("setting up nameboard for " + userName + ". getting user...");
			websocket_client websocket_client = await Basemodal.main_values.api_client.GetClient(userName);
			if (websocket_client == null)
			{
				__instance.Setup(userName, isFriend, title, shieldDefinition);
				return;
			}
			MelonLogger.Msg("the player is a Basemodal user.");
			MelonLogger.Msg("setting displayName to " + websocket_client.user.account_data.displayName);
			__instance.Setup(websocket_client.user.account_data.displayName, false, "potato", shieldDefinition);
		}
	}

	internal static class DLLPatches
	{
		[HarmonyPatch(typeof(UserPermissions), "HasPermission")]
		private static class UserPermissions_HasPermission_Patch
		{
			private static bool Prefix(string permission, ref bool __result)
			{
				if (ForceDebugPermissions && string.Equals(permission, "debug_features", StringComparison.OrdinalIgnoreCase))
				{
					__result = true;
					return false;
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(WithPermissionOnlyAttribute), "Check")]
		private static class WithPermissionOnlyAttribute_Check_Patch
		{
			private static bool Prefix(WithPermissionOnlyAttribute __instance, CommandContext context, ref bool __result)
			{
				if (!ForceDebugPermissions)
				{
					return true;
				}
				string[] roles = __instance.Roles;
				if (roles != null && roles.Any((string r) => string.Equals(r, "debug_features", StringComparison.OrdinalIgnoreCase)))
				{
					__result = true;
					return false;
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(ModsMicMuteManager), "SyncRequest")]
		private static class ModsMicMuteManager_SyncRequest_Patch
		{
			private static bool Prefix(ModsMicMuteManager __instance, IPlayer player, Stream stream, ModMuteArgument argument)
			{
				//IL_002b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0031: Invalid comparison between Unknown and I4
				//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
				//IL_00d1: Unknown result type (might be due to invalid IL or missing references)
				//IL_00f3: Unknown result type (might be due to invalid IL or missing references)
				//IL_00df: Unknown result type (might be due to invalid IL or missing references)
				if (!ForceDebugPermissions)
				{
					return true;
				}
				if (!stream.IsReading || !NetworkSceneManager.IsServer)
				{
					return true;
				}
				if (player == null || player.UserInfo == null || (int)player.UserInfo.MemberStatus >= 2)
				{
					return true;
				}
				stream.SerializeBool(ref argument.isActive);
				stream.SerializeUnsignedInt(ref argument.modPlayerIdentifier);
				try
				{
					object obj = AccessTools.Field(typeof(ModsMicMuteManager), "syncRequest")?.GetValue(__instance);
					((obj != null) ? AccessTools.Method(obj.GetType(), "SendToChunks", new Type[2]
					{
						typeof(ModMuteArgument),
						typeof(IPlayer[])
					}, (Type[])null) : null)?.Invoke(obj, new object[2]
					{
						argument,
						new IPlayer[1] { player }
					});
					if (argument.isActive)
					{
						__instance.ModsMuting[argument.modPlayerIdentifier] = null;
					}
					else
					{
						__instance.ModsMuting.Remove(argument.modPlayerIdentifier);
					}
					return false;
				}
				catch (Exception ex)
				{
					MelonLogger.Warning("[Unified] Mod mic mute dev check bypass failed: " + ex.GetBaseException().Message);
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(MapBoard), "SetupMapPieces")]
		private static class MapBoard_Patch
		{
			private static void Prefix(MapBoard __instance, PlayerMapManager mapManager)
			{
				for (int i = 0; i < 17; i++)
				{
					mapManager.DiscoverPiece(__instance.Map, i);
				}
			}
		}

		[HarmonyPatch(typeof(PlayerMapManager), "GetUndiscoveredMapPieces")]
		private static class PlayerMapManager_AllPiecesPatch
		{
			private static bool Prefix(Map map, ref IEnumerable<int> __result)
			{
				//IL_0001: Unknown result type (might be due to invalid IL or missing references)
				//IL_000c: Expected O, but got Unknown
				if ((Object)map == (Object)null)
				{
					__result = new List<int> { -1 };
					return false;
				}
				__result = Array.Empty<int>();
				return false;
			}
		}

		[HarmonyPatch(typeof(PlayerCommunicationManager), "HandlePlayerMessage")]
		private static class PlayerCommunicationManager_Patch
		{
			private static bool Prefix(IPlayer player, Stream stream, PlayerTextMessage textMessage)
			{
				//IL_004a: Unknown result type (might be due to invalid IL or missing references)
				((PlayerTextMessage)(ref textMessage)).Serialize(player, stream);
				if (stream.IsWriting)
				{
					return false;
				}
				if (string.IsNullOrWhiteSpace(((PlayerTextMessage)(ref textMessage)).Message))
				{
					return false;
				}
				MelonHandler.Mods.OfType<AdonaiUnifiedMod>().FirstOrDefault()?.AppendChatMessage(new ChatMessage(player.UserInfo.Username, ((PlayerTextMessage)(ref textMessage)).Message, Color.cyan));
				MelonLogger.Msg("[chat] " + ((PlayerTextMessage)(ref textMessage)).Message);
				return false;
			}
		}

		[HarmonyPatch(typeof(PickupDock))]
		private static class PickupDockPatches
		{
			[HarmonyPatch]
			private static class CheckAllowedPatch
			{
				private static MethodBase TargetMethod()
				{
					return typeof(PickupDock).GetMethod("CheckAllowed", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, new Type[4]
					{
						typeof(Interactor),
						typeof(Pickup),
						typeof(int),
						typeof(PickupDock).MakeByRefType()
					}, null);
				}

				private static bool Prefix(PickupDock __instance, Interactor interactor, Pickup pickup, int quantity, ref PickupDock finalDock)
				{
					//IL_0006: Unknown result type (might be due to invalid IL or missing references)
					//IL_0010: Unknown result type (might be due to invalid IL or missing references)
					//IL_001a: Expected O, but got Unknown
					//IL_001a: Expected O, but got Unknown
					if ((Object)__instance.PlayerController != (Object)PlayerController.Current)
					{
						AccessTools.PropertySetter(((object)__instance).GetType(), "PlayerController")?.Invoke(__instance, new object[1] { PlayerController.Current });
					}
					return true;
				}
			}

			[HarmonyPatch("CheckForward")]
			[HarmonyPrefix]
			private static bool CheckForwardPrefix(PickupDock __instance, Interactor interactor, ref bool isForced)
			{
				AccessTools.PropertySetter(((object)__instance).GetType(), "PlayerController")?.Invoke(__instance, new object[1] { PlayerController.Current });
				return true;
			}
		}

		[HarmonyPatch(typeof(FogAnimator))]
		private static class FogAnimatorKillPatches
		{
			private static readonly int FadeProp = Shader.PropertyToID("_Fade");

			[HarmonyPatch("Update")]
			[HarmonyPrefix]
			private static bool UpdatePrefix()
			{
				//IL_0015: Unknown result type (might be due to invalid IL or missing references)
				//IL_0020: Expected O, but got Unknown
				RenderSettings.fog = false;
				RenderSettings.fogDensity = 0f;
				if ((Object)RenderSettings.skybox != (Object)null)
				{
					RenderSettings.skybox.SetFloat(FadeProp, 0f);
				}
				return false;
			}

			[HarmonyPrefix]
			[HarmonyPatch("OnEnable")]
			private static bool OnEnablePrefix(FogAnimator __instance)
			{
				RenderSettings.fog = false;
				RenderSettings.fogDensity = 0f;
				((Behaviour)__instance).enabled = false;
				return false;
			}

			[HarmonyPatch("OnDisable")]
			[HarmonyPrefix]
			private static bool OnDisablePrefix()
			{
				RenderSettings.fog = false;
				RenderSettings.fogDensity = 0f;
				return false;
			}
		}

		[HarmonyPatch(typeof(SmoothLocomotion), "MoveToSafety")]
		private static class NoVoidTpPatch
		{
			private static bool Prefix()
			{
				return !Actions.no_void_tp;
			}
		}

		[HarmonyPatch(typeof(BodyController), "UpdateCharacterComponent")]
		private static class BodyController_UpdateCharacterComponent_NoPitchClamp
		{
			private static readonly FieldInfo FinalPitchField = AccessTools.Field(typeof(BodyController), "finalPitch");

			private static readonly FieldInfo FinalHeadingField = AccessTools.Field(typeof(BodyController), "finalHeading");

			private static void Postfix(BodyController __instance)
			{
				//IL_0001: Unknown result type (might be due to invalid IL or missing references)
				//IL_000c: Expected O, but got Unknown
				//IL_0031: Unknown result type (might be due to invalid IL or missing references)
				//IL_003c: Expected O, but got Unknown
				//IL_006b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0076: Expected O, but got Unknown
				//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
				//IL_00bd: Expected O, but got Unknown
				//IL_0083: Unknown result type (might be due to invalid IL or missing references)
				//IL_0088: Unknown result type (might be due to invalid IL or missing references)
				//IL_008f: Unknown result type (might be due to invalid IL or missing references)
				//IL_009e: Unknown result type (might be due to invalid IL or missing references)
				//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
				//IL_00d2: Unknown result type (might be due to invalid IL or missing references)
				//IL_00d6: Unknown result type (might be due to invalid IL or missing references)
				//IL_00db: Unknown result type (might be due to invalid IL or missing references)
				//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
				//IL_00fc: Unknown result type (might be due to invalid IL or missing references)
				//IL_0106: Unknown result type (might be due to invalid IL or missing references)
				if ((Object)__instance == (Object)null || FinalPitchField == null || FinalHeadingField == null)
				{
					return;
				}
				BodyControllerSettings settings = __instance.Settings;
				if (!((Object)settings == (Object)null))
				{
					float num = (float)FinalPitchField.GetValue(__instance);
					float num2 = (float)FinalHeadingField.GetValue(__instance);
					Transform bodyTarget = __instance.BodyTarget;
					if ((Object)bodyTarget != (Object)null)
					{
						float num3 = num * settings.BodyForwardBendMultiplier;
						Quaternion localRotation = bodyTarget.localRotation;
						bodyTarget.localRotation = Quaternion.Euler(num3, ((Quaternion)(ref localRotation)).eulerAngles.y, 0f);
					}
					Transform bodyTransform = __instance.BodyTransform;
					if ((Object)bodyTransform != (Object)null)
					{
						Quaternion val = Quaternion.Euler(num * settings.BodyPitchMultiplier, num2, 0f);
						float num4 = Quaternion.Angle(bodyTransform.localRotation, val);
						float num5 = settings.RotationLerpSpeed.Evaluate(num4);
						bodyTransform.localRotation = Quaternion.Slerp(bodyTransform.localRotation, val, Time.deltaTime * num5);
					}
				}
			}
		}

		[HarmonyPatch(typeof(QuickAccessMenuAction), "InitializeForPlayer")]
		private static class InitializeForPlayer_Patch
		{
			private static bool Prefix(QuickAccessMenuAction __instance)
			{
				//IL_000c: Unknown result type (might be due to invalid IL or missing references)
				//IL_0012: Expected O, but got Unknown
				ShowNames val = (ShowNames)((__instance is ShowNames) ? __instance : null);
				if ((Object)(object)val != (Object)null)
				{
					MelonLogger.Msg("ShowNames InitializeForPlayer triggered");
					typeof(Actions).GetField("showNames", BindingFlags.Static | BindingFlags.NonPublic)?.SetValue(null, val);
					((QuickAccessMenuAction)val).Run(PlayerController.Current.RightController);
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(GameServerInfoExtensions), "JoinServerAsync")]
		private static class JoinServer_Patch
		{
			private static bool Prefix(GameServerInfo gameServer, ref Task<ServerJoinResult> __result)
			{
				if (!IsServerAllowed(gameServer.Identifier))
				{
					QuitForDisallowed("JoinServerAsync id=" + gameServer.Identifier);
					__result = Task.FromResult<ServerJoinResult>(null);
					return true;
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(JoinedServerGameMode), "ReturnToMainMenuOnFailure")]
		private static class ReturnToMainMenuOnFailure_Patch
		{
			private static bool Prefix()
			{
				MelonLogger.Msg("Intercepted ReturnToMainMenuOnFailure");
				return false;
			}
		}

		[HarmonyPatch(typeof(UserRolesUtility), "GetRolesFromIdentityToken", new Type[] { typeof(JwtSecurityToken) })]
		private static class GetRolesFromIdentityToken_Patch
		{
			private static bool Prefix(ref UserRoles __result)
			{
				//IL_0001: Unknown result type (might be due to invalid IL or missing references)
				//IL_0006: Unknown result type (might be due to invalid IL or missing references)
				//IL_000d: Unknown result type (might be due to invalid IL or missing references)
				//IL_0014: Unknown result type (might be due to invalid IL or missing references)
				//IL_001c: Expected O, but got Unknown
				__result = new UserRoles
				{
					IsDeveloper = true,
					IsMember = true,
					IsModerator = true
				};
				return false;
			}
		}

		[HarmonyPatch(typeof(SmoothLocomotionSettings), "CanRun")]
		private static class CanRun_Patch
		{
			private static bool Prefix(ref bool __result)
			{
				__result = true;
				return false;
			}
		}

		[HarmonyPatch(typeof(ClimbingHand), "Process")]
		private static class ClimbingHand_Process_Patch
		{
			private static bool Prefix(ref bool hasStamina)
			{
				hasStamina = true;
				return true;
			}
		}

		[HarmonyPatch(typeof(ClimbingHandler), "Process")]
		private static class ClimbingHandler_Process_Patch
		{
			private static bool Prefix()
			{
				Stat val = null;
				if (PlayerController.Current.Health.StatManager.GetStat(GlobalSettings<ClimbingSettings>.Instance.LeftHandStamina, ref val))
				{
					val.Minimum = val.Maximum;
				}
				Stat val2 = null;
				if (PlayerController.Current.Health.StatManager.GetStat(GlobalSettings<ClimbingSettings>.Instance.RightHandStamina, ref val2))
				{
					val2.Minimum = val2.Maximum;
				}
				return true;
			}
		}

		[HarmonyPatch(typeof(ClimbingHand), "BreakGrip", new Type[] { typeof(bool) })]
		private static class BreakGrip_Patch
		{
			private static bool Prefix(ref bool isVoluntary)
			{
				return isVoluntary;
			}
		}

		[HarmonyPatch(typeof(AmbienceAudio), "Awake")]
		private static class Ambience_Awake_Patch
		{
			private static void Postfix(AmbienceAudio __instance)
			{
				__instance.IsMuted = true;
				__instance.Volume = 0f;
				((SoundSource)__instance).Stop();
			}
		}

		[HarmonyPatch(typeof(SmoothLocomotion), "TryApplyFallDamage")]
		private static class NoFallDamage_Patch
		{
			private static bool Prefix()
			{
				return false;
			}
		}

		[HarmonyPatch(typeof(PersonalSpace), "Awake")]
		private static class PersonalSpace_Awake_Patch
		{
			private static void Postfix(PersonalSpace __instance)
			{
				MelonLogger.Msg("disabling personal space");
				__instance.DisablePersonalSpace();
			}
		}

		[HarmonyPatch(typeof(PlayerUnlockManager), "Check")]
		private static class UnlockManager_Check_Patch
		{
			private static bool Prefix(ref bool __result)
			{
				__result = true;
				return false;
			}
		}

		[HarmonyPatch(typeof(GeneralLoadingObjectManager), "UpdateTipText")]
		private static class LoadingTip_Patch
		{
			private static bool Prefix(GeneralLoadingObjectManager __instance)
			{
				//IL_0006: Unknown result type (might be due to invalid IL or missing references)
				//IL_000b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0037: Unknown result type (might be due to invalid IL or missing references)
				//IL_003e: Expected O, but got Unknown
				//IL_0040: Unknown result type (might be due to invalid IL or missing references)
				//IL_004b: Expected O, but got Unknown
				//IL_0057: Unknown result type (might be due to invalid IL or missing references)
				string text = "Tip: Dont Kill Yourself";
				Color cyan = Color.cyan;
				FieldInfo fieldInfo = AccessTools.Field(((object)__instance).GetType(), "tipTextRenderer");
				object obj = fieldInfo?.GetValue(__instance);
				TextRenderer val = (TextRenderer)((obj is TextRenderer) ? obj : null);
				if ((Object)val != (Object)null)
				{
					val.Text = text;
					val.Color = cyan;
					fieldInfo.SetValue(__instance, val);
				}
				return false;
			}
		}

		[HarmonyPatch(typeof(Gravestone), "Awake")]
		private static class ColorfulGraves_Patch
		{
			private static void Postfix(Gravestone __instance)
			{
				//IL_002a: Unknown result type (might be due to invalid IL or missing references)
				//IL_0030: Expected O, but got Unknown
				//IL_0031: Unknown result type (might be due to invalid IL or missing references)
				//IL_003c: Expected O, but got Unknown
				//IL_00f9: Unknown result type (might be due to invalid IL or missing references)
				//IL_0100: Expected O, but got Unknown
				//IL_0135: Unknown result type (might be due to invalid IL or missing references)
				//IL_013c: Expected O, but got Unknown
				//IL_013e: Unknown result type (might be due to invalid IL or missing references)
				//IL_0149: Expected O, but got Unknown
				//IL_0150: Unknown result type (might be due to invalid IL or missing references)
				//IL_015b: Expected O, but got Unknown
				//IL_0162: Unknown result type (might be due to invalid IL or missing references)
				//IL_016d: Expected O, but got Unknown
				//IL_0172: Unknown result type (might be due to invalid IL or missing references)
				//IL_0177: Unknown result type (might be due to invalid IL or missing references)
				//IL_017b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0180: Unknown result type (might be due to invalid IL or missing references)
				//IL_0184: Unknown result type (might be due to invalid IL or missing references)
				//IL_0189: Unknown result type (might be due to invalid IL or missing references)
				//IL_018d: Unknown result type (might be due to invalid IL or missing references)
				//IL_0195: Unknown result type (might be due to invalid IL or missing references)
				//IL_019e: Unknown result type (might be due to invalid IL or missing references)
				//IL_01a7: Unknown result type (might be due to invalid IL or missing references)
				//IL_01b0: Unknown result type (might be due to invalid IL or missing references)
				//IL_01b2: Unknown result type (might be due to invalid IL or missing references)
				//IL_01b3: Unknown result type (might be due to invalid IL or missing references)
				//IL_01ba: Unknown result type (might be due to invalid IL or missing references)
				//IL_01bc: Unknown result type (might be due to invalid IL or missing references)
				//IL_01c0: Unknown result type (might be due to invalid IL or missing references)
				//IL_01ed: Unknown result type (might be due to invalid IL or missing references)
				//IL_01f2: Unknown result type (might be due to invalid IL or missing references)
				//IL_0216: Unknown result type (might be due to invalid IL or missing references)
				object obj = AccessTools.Field(((object)__instance).GetType(), "revivalBeacon")?.GetValue(__instance);
				GameObject val = (GameObject)((obj is GameObject) ? obj : null);
				if (!((Object)val == (Object)null))
				{
					Transform val2 = val.transform.Find("Audio__BeaconHum");
					if ((Object)(object)val2 != (Object)null)
					{
						((Component)val2).gameObject.SetActive(false);
					}
					Transform val3 = val.transform.Find("Light");
					PlatformLight val4 = (((Object)(object)val3 != (Object)null) ? ((Component)val3).GetComponent<PlatformLight>() : null);
					LightFlicker val5 = (((Object)(object)val3 != (Object)null) ? ((Component)val3).GetComponent<LightFlicker>() : null);
					if ((Object)(object)val5 != (Object)null)
					{
						((Behaviour)val5).enabled = false;
					}
					Transform val6 = val.transform.Find("Windows_PS_2");
					object obj2;
					if ((Object)(object)val6 == (Object)null)
					{
						obj2 = null;
					}
					else
					{
						Transform val7 = val6.Find("PS_Beam");
						obj2 = (((Object)(object)val7 != (Object)null) ? ((Component)val7).GetComponent<ParticleSystem>() : null);
					}
					ParticleSystem val8 = (ParticleSystem)obj2;
					object obj3;
					if ((Object)(object)val6 == (Object)null)
					{
						obj3 = null;
					}
					else
					{
						Transform val9 = val6.Find("PS_BeamTrail");
						obj3 = (((Object)(object)val9 != (Object)null) ? ((Component)val9).GetComponent<ParticleSystem>() : null);
					}
					ParticleSystem val10 = (ParticleSystem)obj3;
					if ((Object)val8 != (Object)null && (Object)val10 != (Object)null && (Object)val4 != (Object)null)
					{
						Color magenta = Color.magenta;
						MainModule main = val8.main;
						MainModule main2 = val10.main;
						MinMaxGradient val11 = default(MinMaxGradient);
						((MinMaxGradient)(ref val11)).color = magenta;
						((MinMaxGradient)(ref val11)).colorMax = magenta;
						((MinMaxGradient)(ref val11)).colorMin = magenta;
						MinMaxGradient val12 = (((MainModule)(ref main)).startColor = val11);
						MinMaxGradient startColor = val12;
						((MainModule)(ref main2)).startColor = startColor;
						((MainModule)(ref main2)).maxParticles = 1000;
						((MainModule)(ref main2)).gravityModifierMultiplier = -1f;
						((MainModule)(ref main2)).startSizeMultiplier = 5f;
						EmissionModule emission = val10.emission;
						((EmissionModule)(ref emission)).rateOverTimeMultiplier = 10f;
						((Behaviour)val4).enabled = true;
						val4.Intensity = 1f;
						val4.Color = magenta;
					}
				}
			}
		}

		[HarmonyPatch(typeof(Connection), "OnDisconnected")]
		private static class Connection_OnDisconnected_Patch
		{
			private static bool Prefix(Connection __instance)
			{
				MelonLogger.Msg("disconnected");
				if (Basemodal.main_values.pressed_disconnect)
				{
					Actions.player_message("successfully disconnected from the server");
					Basemodal.main_values.pressed_disconnect = false;
					return true;
				}
				Basemodal.main_values.in_kicked_dimension = true;
				Basemodal.main_values.is_connected = false;
				Basemodal.main_values.is_loaded_in = false;
				Actions.rejoin_async();
				Actions.player_message("you have been kicked. attempting rejoin...");
				return false;
			}
		}

		[HarmonyPatch(typeof(ServerFeatureSetProvider), "IsServerValid")]
		private static class IsServerValid_Patch
		{
			private static void Postfix(ref bool __result)
			{
				__result = true;
			}
		}

		[HarmonyPatch(typeof(RefreshRequest), "get_Target")]
		private static class RefreshRequest_Target_Get_Patch
		{
			private static void Postfix(ref int? __result)
			{
				__result = null;
			}
		}

		[HarmonyPatch(typeof(CameraStabelizer), "Activate")]
		private static class CameraStabilizer_Patch
		{
			private static bool Prefix()
			{
				return !disable_stabilized_camera;
			}
		}

		[HarmonyPatch(typeof(PlayerDocks), "Filter")]
		private static class PlayerDocks_Filter_Patch
		{
			private static bool Prefix(PlayerDocks __instance)
			{
				AccessTools.Field(typeof(PlayerDocks), "owner")?.SetValue(__instance, PlayerController.Current);
				return true;
			}
		}

		[HarmonyPatch(typeof(TradeVendor), "Initialize")]
		private static class TradeVendor_Initialize_Patch
		{
			private static void Postfix(TradeVendor __instance)
			{
				MelonLogger.Msg("TradeVendor.Initialize patched.");
			}
		}

		[HarmonyPatch(typeof(Interactable))]
		private static class Interactable_IsInteractable_Patch
		{
			[HarmonyPatch("get_IsInteractable")]
			[HarmonyPostfix]
			private static void GetIsInteractablePostfix(ref bool __result)
			{
				__result = true;
			}

			[HarmonyPatch("set_IsInteractable")]
			[HarmonyPrefix]
			private static void SetIsInteractablePrefix(ref bool value)
			{
				value = true;
			}
		}
	}

	private enum WheelPage
	{
		Root,
		Toggles,
		Teleport,
		Speed
	}

	private sealed class PFV3TxtPreviewTag : MonoBehaviour
	{
		public uint Hash;

		public float BaseScale;
	}

	private struct PFV3TxtEntry
	{
		public string Name;

		public uint Hash;

		public float Scale;

		public Vector3 Pos;

		public Vector3 Euler;
	}

	private enum PFV3Mode
	{
		Translate,
		Rotate,
		Scale
	}

	private enum PFV3SelPhase
	{
		Idle,
		DraggingRect,
		Selected,
		Preview
	}

	public class PFV3Captured
	{
		public string Name;

		public string Display;

		public string Id;

		public uint Hash;

		public Vector3 Pos;

		public Quaternion Rot;

		public Vector3 Scale = Vector3.one;

		public Transform Tr;
	}

	private class PFV3Selection
	{
		public bool Active;

		public PFV3SelPhase Phase;

		public readonly List<PFV3Captured> Items = new List<PFV3Captured>();

		public Vector3 A;

		public Vector3 B;

		public Bounds Bounds => new Bounds((A + B) * 0.5f, Vector3.Max(Vector3.one * 0.001f, new Vector3(Mathf.Abs(A.x - B.x), Mathf.Abs(A.y - B.y), Mathf.Abs(A.z - B.z))));

		public void Clear()
		{
			Active = false;
			Phase = PFV3SelPhase.Idle;
			Items.Clear();
		}
	}

	private class PFV3PreviewTx
	{
		public Vector3 Offset;

		public Vector3 EulerAdd;

		public Vector3 Pivot;

		public string Label;

		public List<PFV3ItemState> Src = new List<PFV3ItemState>();

		public Vector3 ScaleMul = Vector3.one;

		public IEnumerable<(uint hash, Vector3 pos, Vector3 eulerYPR)> Emit()
		{
			Vector3 val = default(Vector3);
			foreach (PFV3ItemState item2 in Src)
			{
				Vector3 val2 = item2.Pos - Pivot;
				Vector3 val3 = Quaternion.Euler(EulerAdd) * val2;
				((Vector3)(ref val))._002Ector(val3.x * ScaleMul.x, val3.y * ScaleMul.y, val3.z * ScaleMul.z);
				Vector3 item = Pivot + val + Offset;
				Quaternion val4 = Quaternion.Euler(EulerAdd) * item2.Rot;
				Vector3 eulerAngles = ((Quaternion)(ref val4)).eulerAngles;
				yield return (hash: item2.Hash, pos: item, eulerYPR: new Vector3(eulerAngles.y, eulerAngles.x, eulerAngles.z));
			}
		}
	}

	private sealed class PFV3TxtPreviewState
	{
		public bool Active;

		public string SourcePath;

		public string Label;

		public List<GameObject> Spawned = new List<GameObject>();

		public List<PFV3Captured> Captured = new List<PFV3Captured>();

		public Vector3 Pivot;
	}

	private struct PFV3ItemState
	{
		public string Name;

		public string Display;

		public string Id;

		public uint Hash;

		public Vector3 Pos;

		public Quaternion Rot;

		public Vector3 Scale;

		public Transform Tr;
	}

	private class PFV3Snapshot
	{
		public List<PFV3ItemState> Src = new List<PFV3ItemState>();

		public Vector3 Offset;

		public Vector3 EulerAdd;

		public Vector3 ScaleMul;

		public Vector3 Pivot;

		public PFV3Snapshot()
		{
		}

		public PFV3Snapshot(PFV3PreviewTx tx)
		{
			//IL_0013: Unknown result type (might be due to invalid IL or missing references)
			//IL_0018: Unknown result type (might be due to invalid IL or missing references)
			//IL_001f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0024: Unknown result type (might be due to invalid IL or missing references)
			//IL_002b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0030: Unknown result type (might be due to invalid IL or missing references)
			//IL_0037: Unknown result type (might be due to invalid IL or missing references)
			//IL_003c: Unknown result type (might be due to invalid IL or missing references)
			Offset = tx.Offset;
			EulerAdd = tx.EulerAdd;
			ScaleMul = tx.ScaleMul;
			Pivot = tx.Pivot;
			Src = new List<PFV3ItemState>(tx.Src);
		}
	}

	private struct ServerEntry
	{
		public string Name;

		public string Id;
	}

	private enum PFV3Tool
	{
		Move,
		Clone,
		Stack
	}

	private enum HandSelector
	{
		Left,
		Right,
		Any
	}

	[HarmonyPatch(typeof(HandStore), "Pop")]
	private static class HandStore_Pop_ForceRelease
	{
		private static void Postfix(HandStore __instance, ref bool __result)
		{
			//IL_000c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0017: Expected O, but got Unknown
			try
			{
				Interactor source = __instance.Source;
				if (!__result && (Object)source != (Object)null && source.IsInteracting)
				{
					source.StopInteract(true, true);
					__result = !source.IsInteracting;
				}
			}
			catch
			{
			}
		}
	}

	[HarmonyPatch(typeof(HandStore), "TryPeek")]
	private static class HandStore_TryPeek_Lenient
	{
		private static void Postfix(HandStore __instance, ref bool __result, ref Pickup pickup)
		{
			//IL_002c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0032: Expected O, but got Unknown
			//IL_0033: Unknown result type (might be due to invalid IL or missing references)
			//IL_003e: Expected O, but got Unknown
			if (__result)
			{
				return;
			}
			try
			{
				Interactor source = __instance.Source;
				Interactable val = (((Object)(object)source != (Object)null) ? source.InteractingWith : null);
				Pickup val2 = (Pickup)((val is Pickup) ? val : null);
				if ((Object)val2 != (Object)null)
				{
					pickup = val2;
					__result = true;
				}
			}
			catch
			{
			}
		}
	}

	[Serializable]
	public class SavedServer
	{
		public string Name;

		public string Id;
	}

	[Serializable]
	public class SaveBlob
	{
		public List<SavedServer> Servers = new List<SavedServer>();

		public string ApiToken = "";
	}

	private struct PrefabEntry
	{
		public string Name;

		public uint Hash;
	}

	private class EditorFlyCam : MonoBehaviour
	{
		public float speed = 8f;

		public float sprint = 3f;

		public float lookSens = 0.12f;

		public bool lookFrozen;

		public bool requireRightMouse;

		private Vector2 _yawPitch;

		private void OnEnable()
		{
			ApplyCursor();
		}

		private void OnDisable()
		{
			Cursor.lockState = (CursorLockMode)0;
			Cursor.visible = true;
		}

		private void ApplyCursor()
		{
			if (lookFrozen)
			{
				Cursor.lockState = (CursorLockMode)0;
				Cursor.visible = true;
			}
			else
			{
				Cursor.lockState = (CursorLockMode)1;
				Cursor.visible = false;
			}
		}

		private void Update()
		{
			//IL_00f3: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f8: Unknown result type (might be due to invalid IL or missing references)
			//IL_0106: Unknown result type (might be due to invalid IL or missing references)
			//IL_0107: Unknown result type (might be due to invalid IL or missing references)
			//IL_010c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0111: Unknown result type (might be due to invalid IL or missing references)
			//IL_0070: Unknown result type (might be due to invalid IL or missing references)
			//IL_0075: Unknown result type (might be due to invalid IL or missing references)
			//IL_0084: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a5: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
			//IL_011f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0120: Unknown result type (might be due to invalid IL or missing references)
			//IL_0125: Unknown result type (might be due to invalid IL or missing references)
			//IL_012a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0138: Unknown result type (might be due to invalid IL or missing references)
			//IL_0139: Unknown result type (might be due to invalid IL or missing references)
			//IL_013e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0143: Unknown result type (might be due to invalid IL or missing references)
			//IL_0151: Unknown result type (might be due to invalid IL or missing references)
			//IL_0152: Unknown result type (might be due to invalid IL or missing references)
			//IL_0157: Unknown result type (might be due to invalid IL or missing references)
			//IL_015c: Unknown result type (might be due to invalid IL or missing references)
			//IL_016a: Unknown result type (might be due to invalid IL or missing references)
			//IL_016b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0170: Unknown result type (might be due to invalid IL or missing references)
			//IL_0175: Unknown result type (might be due to invalid IL or missing references)
			//IL_01c1: Unknown result type (might be due to invalid IL or missing references)
			//IL_01cc: Unknown result type (might be due to invalid IL or missing references)
			//IL_01cd: Unknown result type (might be due to invalid IL or missing references)
			//IL_01d3: Unknown result type (might be due to invalid IL or missing references)
			//IL_01dd: Unknown result type (might be due to invalid IL or missing references)
			//IL_01e2: Unknown result type (might be due to invalid IL or missing references)
			Keyboard current = Keyboard.current;
			Mouse current2 = Mouse.current;
			if (current != null && current2 != null)
			{
				if (((ButtonControl)current.minusKey).wasPressedThisFrame || ((ButtonControl)current.numpadMinusKey).wasPressedThisFrame)
				{
					lookFrozen = !lookFrozen;
					ApplyCursor();
				}
				if (!lookFrozen && (!requireRightMouse || current2.rightButton.isPressed))
				{
					Vector2 val = ((InputControl<Vector2>)(object)((Pointer)current2).delta).ReadValue();
					_yawPitch.x += val.x * lookSens;
					_yawPitch.y = Mathf.Clamp(_yawPitch.y - val.y * lookSens, -89f, 89f);
					((Component)this).transform.rotation = Quaternion.Euler(_yawPitch.y, _yawPitch.x, 0f);
				}
				Vector3 val2 = Vector3.zero;
				if (((ButtonControl)current.wKey).isPressed)
				{
					val2 += Vector3.forward;
				}
				if (((ButtonControl)current.sKey).isPressed)
				{
					val2 += Vector3.back;
				}
				if (((ButtonControl)current.aKey).isPressed)
				{
					val2 += Vector3.left;
				}
				if (((ButtonControl)current.dKey).isPressed)
				{
					val2 += Vector3.right;
				}
				if (((ButtonControl)current.spaceKey).isPressed)
				{
					val2 += Vector3.up;
				}
				float num = speed * ((((ButtonControl)current.leftShiftKey).isPressed || ((ButtonControl)current.rightShiftKey).isPressed) ? sprint : 1f);
				if (((Vector3)(ref val2)).sqrMagnitude > 1f)
				{
					((Vector3)(ref val2)).Normalize();
				}
				Transform transform = ((Component)this).transform;
				transform.position += ((Component)this).transform.TransformDirection(val2) * num * Time.unscaledDeltaTime;
			}
		}
	}

	public class ChatMessage
	{
		public string Sender;

		public string Text;

		public Color Color;

		public ChatMessage(string sender, string text, Color color)
		{
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_0016: Unknown result type (might be due to invalid IL or missing references)
			Sender = sender;
			Text = text;
			Color = color;
		}
	}

	[HarmonyPatch(typeof(GameModeManager), "JoinServer", new Type[] { typeof(GameServerInfo) })]
	private static class JoinServerInfoPatch
	{
		private static bool Prefix(GameServerInfo server)
		{
			if (server == null || !IsServerAllowed(server.Identifier))
			{
				QuitForDisallowed("JoinServer(GameServerInfo) id=" + ((server != null) ? server.Identifier : (-1)));
				return true;
			}
			return true;
		}
	}

	[HarmonyPatch]
	private static class JoinServerIdPatch
	{
		private static MethodBase TargetMethod()
		{
			return AccessTools.Method(typeof(GameModeManager), "JoinServer", new Type[1] { typeof(int) }, (Type[])null);
		}

		private static bool Prepare()
		{
			return TargetMethod() != null;
		}

		private static bool Prefix(int serverId)
		{
			if (!IsServerAllowed(serverId))
			{
				QuitForDisallowed("JoinServer(int) id=" + serverId);
				return true;
			}
			return true;
		}
	}

	private Vector2 _camUISize = new Vector2(210f, 148f);

	private Vector3 _camUIOffsetWorld = new Vector3(0.06f, 0.05f, 0.1f);

	private bool _camUIOpen;

	private bool _camWasActive;

	private static readonly List<int> _camTimerOptions = new List<int>();

	private static bool _camTimerInit = false;

	private static float _camTimerSec = 0f;

	private GameObject boardClone;

	private bool minimapVisible = true;

	private Vector3 minimapOffset = new Vector3(0.078f, -0.02f, 0.118f);

	private Vector3 minimapScale = new Vector3(0.029f, 0.029f, 0.029f);

	private readonly Vector3 boardHiddenPos = new Vector3(-0.31f, -100f, 0.569f);

	private GameObject minimapOutline;

	private float outlineWidth = 2.2578f;

	private float outlineHeight = 2.0402f;

	private Vector3 outlineOffset = new Vector3(0.0661f, 1.669f, -0.0098f);

	private Vector2 mapRefA = new Vector2(-0.7f, 1f);

	private Vector2 mapRefB = new Vector2(0f, 2.3f);

	private Vector2 mapRefC = new Vector2(1f, 1.3f);

	private Vector2 worldRefA = new Vector2(-1425.5f, -347.3f);

	private Vector2 worldRefB = new Vector2(-849.7f, 737.1f);

	private Vector2 worldRefC = new Vector2(62.2f, -102.2f);

	private Matrix4x4 mapTransform;

	private bool mapTransformReady;

	private const float MenuToggleDoubleWindow = 0.35f;

	private const Key MenuToggleKey = (Key)94;

	private bool _menuVisible = true;

	private MenuTab _menuTab;

	private Vector2 _menuScrollConnect;

	private Vector2 _menuScrollGeneral;

	private Vector2 _menuScrollUtilities;

	private Vector2 _voiceScroll;

	private Rect _menuBounds = Rect.zero;

	private string _offsetRightText = "-0.5";

	private string _offsetFwdText = "1.0";

	private BodyControllerSettings[] _cachedBodyBendSettings;

	private FieldInfo _bodyBendField;

	private string _spawnPrefab = "";

	private string _spawnArgs = "";

	private string _progressCaveDepth = "6";

	private string _timeSetInput = "12:00";

	private string _timeFutureMinutes = "60";

	private string _heatMultiplierText = "1.0";

	private string _settingsToggleName = "";

	private string _settingsTarget = "game";

	private string _settingsName = "";

	private string _settingsValue = "";

	private const float TeleportSphereRadius = 20f;

	private const float TeleportSphereSpacing = 1f;

	private const float TeleportSphereThickness = 0.5f;

	private const float TeleportSphereInterval = 1f;

	private const float TeleportSpherePillarHeight = 4f;

	private const float TeleportSpherePillarSpacing = 0.5f;

	private const float TeleportConeHeight = 18f;

	private const float TeleportConeRadius = 10f;

	private const float TeleportConeSpacing = 1f;

	private const float TeleportStarOuterRadius = 12f;

	private const float TeleportStarInnerRadius = 6f;

	private const float TeleportStarSpacing = 1f;

	private const float TeleportStarThickness = 1f;

	private const float TeleportStarLayerSpacing = 0.5f;

	private const float TeleportRingRadius = 12f;

	private const float TeleportRingSpacing = 0.6f;

	private const float TeleportRingThickness = 1f;

	private const float TeleportRingLayerSpacing = 0.5f;

	private const float TeleportRingArenaInterval = 0.12f;

	private const float TeleportRingArenaShrinkSpeed = 0.5f;

	private const float TeleportRingArenaMinRadius01 = 0.08f;

	private const float TeleportCubeSize = 16f;

	private const float TeleportCubeSpacing = 1f;

	private const float TeleportHelixHeight = 16f;

	private const float TeleportHelixRadius = 6f;

	private const float TeleportHelixTurns = 3f;

	private const float TeleportHelixSpacing = 0.5f;

	private static List<Vector3> _teleportSphereOffsets;

	private static List<Vector3> _teleportConeOffsets;

	private static List<Vector3> _teleportStarOffsets;

	private static List<Vector3> _teleportRingOffsets;

	private static List<Vector3> _teleportCubeOffsets;

	private static List<Vector3> _teleportHelixOffsets;

	private bool _teleportSphereEnabled;

	private bool _teleportConeEnabled;

	private bool _teleportStarEnabled;

	private bool _teleportRingEnabled;

	private bool _teleportRingArenaEnabled;

	private bool _teleportCubeEnabled;

	private bool _teleportHelixEnabled;

	private float _teleportSphereNextTime;

	private Vector3 _teleportSphereCenter;

	private bool _teleportSphereCenterSet;

	private float _teleportRingArenaRadius;

	private float _teleportRingArenaNextTime;

	private float _teleportRingArenaLastTime;

	private float _teleportShapeScale = 1f;

	private const float InvisScanInterval = 0.5f;

	private bool _invisEnabled;

	private PlayerController _invisPlayer;

	private Transform _invisRoot;

	private float _invisNextScanAt;

	private readonly Dictionary<Renderer, bool> _invisOriginalStates = new Dictionary<Renderer, bool>();

	private Camera thirdPersonCam;

	private bool thirdpersonMode;

	private Vector3 thirdpersonOffset = new Vector3(0f, 0.5f, -3f);

	private bool snapFollowPrimed;

	private bool snapFollowActive;

	private Transform snapFollowTarget;

	private Transform snapFollowHeadAnchor;

	private Vector3 snapFollowOffset = new Vector3(0f, 0f, -1.5f);

	private const float PartnerHoldGripAxisThreshold = 0.6f;

	private const float PartnerHoldSkeletonGripThreshold = 0.8f;

	private const float PartnerHoldSlotRadius = 0.5f;

	private const float PartnerHoldVelocitySmoothing = 14f;

	private const float PartnerThrowDurationSeconds = 0.25f;

	private const float PartnerThrowDecay = 8f;

	private const float DefaultOrbitRadius = 1.2f;

	private const float MinOrbitRadius = 0.3f;

	private const float MaxOrbitRadius = 4f;

	private const float RadiusGrowPerSec = 1.8f;

	private const float AlignLerp = 18f;

	private static readonly string[] PartnerSlotNeedles = new string[4] { "Tool Slot Right", "Tool Slot Left", "Pouch Slot Right", "Pouch Slot Left" };

	private static readonly string LeftHandBone = "hand_male_body_default_L_01";

	private static readonly string RightHandBone = "hand_male_body_default_R_01";

	private bool _partnerHoldPrimed;

	private Transform _partnerRoot;

	private Transform _partnerLeftHand;

	private Transform _partnerRightHand;

	private Transform[] _mySlots;

	private readonly PartnerHoldState _hold = new PartnerHoldState();

	private readonly PartnerOrbitState _orbit = new PartnerOrbitState();

	private bool _throwActive;

	private Vector3 _throwVelocity;

	private float _throwTimeLeft;

	public int maxESPMarkers = 400;

	public bool espParentToTarget;

	public bool espCopyRotation;

	public float espWorldSize = 0.22f;

	public bool espShowLabels = true;

	public int espLabelMaxLen = 28;

	public float espLabelYOffset = 0.28f;

	public float espLabelBaseSize = 0.4f;

	public Color espColor = new Color(0.33f, 0f, 0.55f);

	public bool espOnePerRoot = true;

	public string espInput = "";

	public float espRefreshInterval = 1f;

	public bool espEnabled;

	private Material espOverlayMaterial;

	private readonly List<GameObject> espMarkers = new List<GameObject>();

	private readonly List<MarkerInfo> _markers = new List<MarkerInfo>();

	private float _nextRefreshTime;

	private bool _isSpinning;

	private Camera _spinningCamera;

	private Vector3 _lockedLeftHandPos;

	private Quaternion _lockedLeftHandRot;

	private Vector3 _lockedRightHandPos;

	private Quaternion _lockedRightHandRot;

	private Quaternion _lockedBodyRotation;

	private const float SpinningSpeed = 100f;

	private const float SpinningCameraHeight = 10f;

	private bool _chatOverlayVisible = true;

	private bool _controlActive;

	private bool _showUI = true;

	private bool _chatFocusPending;

	private GUIStyle _sidebarBtn;

	private GUIStyle _btn;

	private GUIStyle _label;

	private GUIStyle _small;

	private GUIStyle _h1;

	private GUIStyle _h2;

	private GUIStyle _h3;

	private MeshRenderer _globalMapRenderer;

	internal GameObject _globalMapGO;

	private bool _mapFollowEnabled = true;

	private bool _sceneReady;

	private GUIStyle _card;

	private GUIStyle _topTab;

	private Texture2D _texPanel;

	private Texture2D _texDivider;

	private string _statusLeft = "";

	private string _statusRight = "";

	private string _customTeleportInput = "";

	private bool _prevLeftGrab;

	private bool _prevRightGrab;

	private BasemodalApiInteractor _api;

	private bool _bhEnabled;

	private Vector3 _bhOffset = new Vector3(0f, 0.12f, 0.18f);

	private bool _awaitingToken;

	private float _currentSpeedCache = -1f;

	private readonly HashSet<int> _knownPlayers = new HashSet<int>();

	private float _playerPollNextTime;

	private bool _apiRegisteredWithServer;

	private int _apiRegisteredServerId = -1;

	private Task _apiJoinTask = Task.CompletedTask;

	private readonly List<PlayerController> _lockBuffer = new List<PlayerController>();

	private int _lockIndex = -1;

	private InputAction _xrRStick;

	private InputAction _xrRStickClick;

	private Vector2 _xrRStickAxis;

	private InputAction _xrLStick;

	private float _menuToggleLastClick = -10f;

	private bool _wheelVisible;

	private float _wheelHoverStart = -1f;

	private int _wheelHoverIndex = -1;

	private InputAction _chatHotkey;

	private InputAction _chatSendKey;

	private RadialMenu3D _wheel;

	private GameObject _wheelGO;

	private bool _csEnabled;

	private bool _smiteEnabled;

	private bool _bladeHeldFxEnabled;

	private const bool ForceBladeSkillsAlwaysOn = true;

	private bool _autoHarvestEnabled;

	private float _autoHarvestTimer;

	private float _autoHarvestResetTimer;

	private bool _autoReviveEnabled;

	private bool _autoReviveInProgress;

	private float _autoReviveNextTime;

	private static readonly Vector3 AutoReviveHandOffset = new Vector3(-0.1f, -0.02f, 0.3f);

	private static readonly Vector3 AutoReviveHandEuler = new Vector3(-30f, -37.1f, 72.4f);

	private bool _modMicMuteActive;

	private bool _impactDebugSubscribed;

	private bool _damageDebugEnabled = true;

	private const float AutoHarvestInterval = 0f;

	private const float AutoHarvestResetSeconds = 30f;

	private const float WHEEL_HOVER_SELECT_TIME = 0.13f;

	private const float WHEEL_INNER = 0.045f;

	private const float WHEEL_OUTER = 0.135f;

	private const int WHEEL_SEGMENTS_PER_SLICE = 12;

	private WheelPage _wheelPage;

	private int _tpPageIndex;

	private const int TP_PER_PAGE = 5;

	private const float HAND_SNATCH_RADIUS = 0.45f;

	private const float LOOK_RAY_DISTANCE = 1000f;

	private const float LOOK_RAY_RADIUS = 0.12f;

	private const float LOOK_EMISSION_INTENSITY = 2.2f;

	private static readonly RaycastHit[] _lookRayHits = (RaycastHit[])(object)new RaycastHit[64];

	private static readonly string[] _lookIgnoreNameFragments = new string[1] { "Dock" };

	private static readonly Color LOOK_HIGHLIGHT_COLOR = new Color(0.05f, 0.32f, 0.05f, 1f);

	private Pickup _lookCurrentPickup;

	private readonly Dictionary<Renderer, Material[]> _lookOriginalMats = new Dictionary<Renderer, Material[]>();

	private readonly Dictionary<Renderer, MaterialPropertyBlock> _lookOriginalMPBs = new Dictionary<Renderer, MaterialPropertyBlock>();

	private Material _lookFallbackHighlightMat;

	private float _pawgleNextScanTime;

	private const float PAWGLE_SCAN_INTERVAL = 1f;

	private ClientWebSocket _ws;

	private CancellationTokenSource _wsCts;

	private Task _wsRecvTask;

	private bool _wsAlive;

	private volatile bool _wsAuthenticated;

	private readonly Queue<ChatMessage> _pendingChat = new Queue<ChatMessage>();

	private readonly object _pendingLock = new object();

	private string _pfv3BlueprintDir;

	private string _pfv3BlueprintDirCached;

	private string[] _pfv3TxtFiles = Array.Empty<string>();

	private int _pfv3TxtIndex = -1;

	private bool _pfv3SaveNearby;

	private string _pfv3SaveName = "";

	private bool _pfv3ShowFileMenu;

	private Vector2 _pfv3FileMenuScroll;

	private bool _pfv3TxtPreviewActive;

	private Transform _pfv3TxtPreviewRoot;

	private Vector3 _pfv3PreviewScale = Vector3.one;

	private Vector3 _pfv3StartScale = Vector3.one;

	private float _pfv3ScaleSnap = 0.1f;

	private float _pfv3ScaleRefLen = 1f;

	private string _pfv3CmdLine = "";

	private readonly PFV3Selection _pfv3Sel = new PFV3Selection();

	private readonly Stack<PFV3Snapshot> _pfv3Undo = new Stack<PFV3Snapshot>();

	private readonly Stack<PFV3Snapshot> _pfv3Redo = new Stack<PFV3Snapshot>();

	private PFV3Mode _pfv3Mode;

	private bool _pfv3EditorOn;

	private bool _pfv3TabFocus;

	private static Material _pfv3LineMat;

	private float _pfv3GizmoSize = 1f;

	private const float PFV3_PICK_RADIUS_SCREEN = 16f;

	private const float PFV3_AXIS_LEN = 1.45f;

	private const float PFV3_ARC_RADIUS = 1.2f;

	private bool _pfv3Dragging;

	private int _pfv3ActiveAxis = -1;

	private Vector3 _pfv3DragStartWorld;

	private Vector3 _pfv3DragStartOffset;

	private Vector3 _pfv3StartEulerAdd;

	private Vector2 _pfv3MouseStart;

	private Vector3 _pfv3Pivot;

	private Vector3 _pfv3PreviewOffset = Vector3.zero;

	private Vector3 _pfv3PreviewEuler = Vector3.zero;

	private float _pfv3MoveSnap = 0.25f;

	private float _pfv3RotateSnap = 15f;

	private readonly List<PFV3Captured> _pfv3Scan = new List<PFV3Captured>();

	private readonly Dictionary<string, uint> _pfv3Name2Hash = new Dictionary<string, uint>(StringComparer.OrdinalIgnoreCase);

	private readonly Dictionary<uint, string> _pfv3Hash2Name = new Dictionary<uint, string>();

	private string _pfv3CsvPath = "";

	private Vector3 _pfv3StackStep = new Vector3(1f, 1f, 1f);

	private int _pfv3StackX = 1;

	private int _pfv3StackY = 1;

	private int _pfv3StackZ = 1;

	private bool _pfv3Kinematic;

	private bool _pfv3ScanDrop;

	private bool _pfv3SelDrop;

	private Vector2 _pfv3ScanDropScroll;

	private Vector2 _pfv3SelDropScroll;

	private string _pfv3ScanFilter = "";

	private int _pfv3SelFocus = -1;

	private Camera _pfv3EditorCam;

	private Camera _pfv3PrevMainCam;

	private EditorFlyCam _pfv3EditorCtl;

	private string _pfv3HoverName;

	private uint _pfv3HoverHash;

	private const float PFV3_TOPBAR_H = 68f;

	private const float PFV3_RIGHT_W = 360f;

	private string _pfv3SnapToId = "";

	private float _pfv3NextSendAt;

	private const float PFV3_SEND_HZ = 10f;

	private Vector3 _pfv3LastEulerSent;

	private bool _pfv3HaveEulerSent;

	private Transform _pfv3HoverTr;

	private Vector3 _pfv3HoverPos;

	private bool _pfv3RectDragging;

	private Vector2 _pfv3RectStart;

	private Vector2 _pfv3RectNow;

	private Texture2D _pfv3RectFill;

	private Texture2D _pfv3RectBorder;

	private bool _pfv3ShowSelectionBox;

	private bool _pfv3ShowRuler;

	private bool _pfv3ShowPlanes;

	private readonly List<Vector3> _pfv3RulerPts = new List<Vector3>();

	private bool _pfv3RulerSnapToSurface = true;

	private float _pfv3PlaneSize = 2f;

	private int _pfv3PlaneGrid = 8;

	private bool _pfv3SpawnPreviewActive;

	private string _pfv3SpawnPreviewFile;

	private PFV3TxtPreviewState _pfv3TxtPreview = new PFV3TxtPreviewState();

	private readonly List<ServerEntry> _saved = new List<ServerEntry>();

	private static readonly HashSet<int> _joinableServerIds = new HashSet<int>();

	private int _selServer = -1;

	private string _srvId = "";

	private string _srvName = "";

	private string _connectErr = "";

	private bool _dropdownOpen;

	private Vector2 _dropScroll;

	private bool _pfv3LocalAxes;

	private int _pfv3KeyAxis = -1;

	private float _pfv3NudgeStep = 0.1f;

	private bool _pfv3KeepExisting;

	private PFV3Tool _pfv3Tool;

	private bool _lightOn;

	private static readonly HashSet<int> _defaultJoinableServerIds = new HashSet<int>();

	private static readonly Dictionary<int, string> AllowedServers = new Dictionary<int, string>
	{
		{ 1263806257, "Domain private" },
		{ 928718057, "Isle of Myths" },
		{ 1737100051, "Shining Sun Empire" },
		{ 1704646669, "Eldervale [Races Spirits Guilds]" }
	};

	private Rect _simpleUiRect = new Rect(16f, 16f, 360f, 520f);

	private Dictionary<string, Vector3> _teleports = new Dictionary<string, Vector3>
	{
		["PlayerSpawn!"] = new Vector3(-691.062f, 129.298f, 72.524f),
		["BlackSmith"] = new Vector3(-737.8415f, 134.16968f, 8.566498f),
		["Carpentry"] = new Vector3(-748.00446f, 130.12466f, 87.894f),
		["Tavern"] = new Vector3(-801.8606f, 135.77469f, -5.0390005f),
		["TownHall"] = new Vector3(-900.746f, 162.46484f, 109.066f),
		["Home"] = new Vector3(-301.186f, 128.31999f, -132.654f),
		["Cave127"] = new Vector3(-703.60114f, -1798.563f, -1100.5371f),
		["Tower"] = new Vector3(-925.688f, 797.5f, -1760.318f),
		["PlotOutsideMap"] = new Vector3(-2027.221f, 211.037f, 2030.3f),
		["MountainTopCamp"] = new Vector3(-407.345f, 172.31299f, 144.429f),
		["MiningHouse"] = new Vector3(-842.201f, 143.804f, 36.987f),
		["CraftingHouse"] = new Vector3(-796.3364f, 143.5237f, 103.409004f),
		["Shops"] = new Vector3(-804.7894f, 135.171f, 50.216f),
		["HebiousCamp1"] = new Vector3(-301.186f, 128.31999f, -132.654f),
		["Trap"] = new Vector3(-1145.411f, 496.216f, -1261.214f),
		["FleaMarket"] = new Vector3(-1242.494f, 765.977f, 864.094f),
		["CarnivalLand"] = new Vector3(1564.192f, 239.364f, 201.379f),
		["BlindPlace"] = new Vector3(-717.259f, 105.237f, 0.2730019f),
		["Forging"] = new Vector3(-642.476f, 151.46799f, 20.122f),
		["Mining"] = new Vector3(-825.0035f, 180.54062f, -60.31f),
		["Melee"] = new Vector3(-1403.143f, 177.07399f, 161.848f),
		["Range"] = new Vector3(-834.661f, 214.09401f, 384.151f),
		["Climbing"] = new Vector3(-873.467f, 528.38696f, -1761.549f),
		["Woodcutting"] = new Vector3(-405.492f, 164.13399f, 19.685999f)
	};

	private string[] _tpOptions;

	private int _tpSelected = -1;

	private bool _tpDrop;

	private string _savePath;

	private SaveBlob _save = new SaveBlob();

	private float _scanRadius = 20f;

	private readonly List<PrefabEntry> _scanned = new List<PrefabEntry>();

	private string[] _options = Array.Empty<string>();

	private List<int> _sel = new List<int> { 0 };

	private bool _drop;

	private Vector2 _dropSc;

	private float _moveStep = 1f;

	private float _rotateStep = 15f;

	private string _cmd = "";

	public static GameObject brushSphere;

	public static bool brushModeActive = false;

	private float brushSize = 3f;

	private bool isRunningPickupRoutine;

	private readonly List<GameObject> collectedItems = new List<GameObject>();

	private readonly Dictionary<GameObject, Material[]> originalMaterials = new Dictionary<GameObject, Material[]>();

	private Material highlightMaterial;

	private int _pfv3HoveredAxis = -1;

	private Vector3 _pfv3DragPlaneNormal = Vector3.forward;

	private float _pfv3ScaleStartRadius = 1f;

	private int _pfv3RayMask = -1;

	private Vector2 _pfv3RightScroll;

	private static readonly IFormatProvider Inv = CultureInfo.InvariantCulture;

	private bool _terrainLowered;

	private readonly List<ChatMessage> _chat = new List<ChatMessage>();

	private Vector2 _chatScroll;

	private string _input = "";

	private Rect _chatRect = new Rect(10f, 10f, 400f, 300f);

	private GUIStyle _chatStyle;

	private GUIStyle _chatPanelStyle;

	private GUIStyle _chatSenderStyle;

	private GUIStyle _chatInputStyle;

	private bool _chatNewMessage;

	private bool _chatTyping;

	private bool _chatSending;

	private float _chatFadeAlpha = 1f;

	private float _chatLastActivity;

	private const int ChatHistoryLimit = 150;

	private const float ChatFadeDelay = 6f;

	private const float ChatFadeDuration = 2.5f;

	private string _tokenInput = "";

	private bool _tokenFocusPending;

	private bool _tokenSubmitting;

	public static ServerJoinResult PreJoinResult { get; set; } = null;

	public static bool disable_stabilized_camera { get; set; } = false;

	internal static bool ForceDebugPermissions { get; set; } = true;

	private Camera PFV3_Cam => PFV3_ActiveCamera;

	private Transform PFV3_Head => Actions.find_head();

	private Camera PFV3_ActiveCamera
	{
		get
		{
			//IL_0006: Unknown result type (might be due to invalid IL or missing references)
			//IL_0011: Expected O, but got Unknown
			//IL_004a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0050: Expected O, but got Unknown
			object obj;
			if (!((Object)_pfv3EditorCam != (Object)null) || !((Behaviour)_pfv3EditorCam).enabled)
			{
				PlayerController current = PlayerController.Current;
				obj = (((Object)(object)current != (Object)null) ? current.Camera : null);
				if (obj == null)
				{
					return Camera.main;
				}
			}
			else
			{
				obj = _pfv3EditorCam;
			}
			return (Camera)obj;
		}
	}

	private string PFV3_BlueprintDir
	{
		get
		{
			if (string.IsNullOrEmpty(_pfv3BlueprintDirCached))
			{
				string text = Path.Combine(Application.dataPath, "Mods", "Prefabs");
				try
				{
					Directory.CreateDirectory(text);
				}
				catch
				{
				}
				_pfv3BlueprintDirCached = text;
			}
			return _pfv3BlueprintDirCached;
		}
	}

	private void CameraUI_Tick()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		HandCamera current = HandCamera.Current;
		bool flag = (Object)current != (Object)null && ((Behaviour)current).isActiveAndEnabled;
		if (flag && !_camWasActive)
		{
			_camUIOpen = true;
			_camTimerInit = false;
		}
		_camWasActive = flag;
		if (!flag)
		{
			_camUIOpen = false;
		}
		if (_camUIOpen)
		{
			Keyboard current2 = Keyboard.current;
			if (current2 != null && ((ButtonControl)current2.escapeKey).wasPressedThisFrame)
			{
				_camUIOpen = false;
			}
		}
	}

	private void DrawCameraPanel()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Expected O, but got Unknown
		//IL_005d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		//IL_009a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ff: Unknown result type (might be due to invalid IL or missing references)
		//IL_017c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_019a: Expected O, but got Unknown
		//IL_02ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d9: Expected O, but got Unknown
		HandCamera current = HandCamera.Current;
		if ((Object)current == (Object)null || !((Behaviour)current).isActiveAndEnabled || !_camUIOpen)
		{
			return;
		}
		PlayerController current2 = PlayerController.Current;
		Camera val = (((Object)(object)current2 != (Object)null) ? current2.Camera : null) ?? Camera.main;
		if ((Object)val == (Object)null)
		{
			return;
		}
		Vector3 val2 = ((Component)current).transform.TransformPoint(_camUIOffsetWorld);
		Vector3 val3 = val.WorldToScreenPoint(val2);
		if (val3.z <= 0f)
		{
			return;
		}
		float x = _camUISize.x;
		float y = _camUISize.y;
		float num = val3.x - x * 0.5f;
		float num2 = (float)Screen.height - val3.y - y * 0.5f;
		float num3 = Mathf.Clamp(num, 6f, (float)Screen.width - x - 6f);
		num2 = Mathf.Clamp(num2, 6f, (float)Screen.height - y - 6f);
		GUILayout.BeginArea(new Rect(num3, num2, x, y), _card ?? GUI.skin.box);
		GUILayout.BeginVertical(_card ?? GUI.skin.box, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("\ud83d\udcf7 Hand Cam", _h3 ?? GUI.skin.label, Array.Empty<GUILayoutOption>());
		GUILayout.FlexibleSpace();
		GUILayout.EndHorizontal();
		GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 4f), (Texture)(_texDivider ?? Texture2D.whiteTexture));
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.BeginVertical(Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Screenshot", _btn ?? GUI.skin.button, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			SafeInvoke(delegate
			{
				HandCamera current4 = HandCamera.Current;
				if ((Object)(object)current4 != (Object)null)
				{
					current4.TakeScreenshot();
				}
			});
		}
		if (GUILayout.Button("Flip", _btn ?? GUI.skin.button, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			SafeInvoke(delegate
			{
				HandCamera current4 = HandCamera.Current;
				if ((Object)(object)current4 != (Object)null)
				{
					current4.FlipCamera();
				}
			});
		}
		if (GUILayout.Button("Stream", _btn ?? GUI.skin.button, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			SafeInvoke(delegate
			{
				HandCamera current4 = HandCamera.Current;
				if ((Object)(object)current4 != (Object)null)
				{
					current4.ToggleStreaming();
				}
			});
		}
		HandCamera current3 = HandCamera.Current;
		if ((Object)current3 == (Object)null)
		{
			GUILayout.Label("No hand camera found.", _small ?? GUI.skin.label, Array.Empty<GUILayoutOption>());
			GUILayout.EndVertical();
			GUILayout.EndHorizontal();
			GUILayout.EndVertical();
			GUILayout.EndArea();
			return;
		}
		ScanCameraTimerOptionsIfNeeded(current3);
		if (!_camTimerInit)
		{
			int num4 = TryGetCurrentTimerSeconds(current3);
			_camTimerSec = Mathf.Max(0, num4);
			_camTimerInit = true;
		}
		int num5 = ((_camTimerOptions.Count > 0) ? _camTimerOptions[_camTimerOptions.Count - 1] : 10);
		GUILayout.Space(4f);
		GUILayout.Label($"Stay: {Mathf.RoundToInt(_camTimerSec)}s", _small ?? GUI.skin.label, Array.Empty<GUILayoutOption>());
		float num6 = GUILayout.HorizontalSlider(_camTimerSec, 0f, (float)num5, Array.Empty<GUILayoutOption>());
		int num7 = Mathf.RoundToInt(_camTimerSec);
		int num8 = Mathf.RoundToInt(num6);
		if (num8 != num7)
		{
			_camTimerSec = num6;
			SetCameraTimerToNearest(current3, num8);
		}
		GUILayout.EndVertical();
		GUILayout.EndHorizontal();
		GUILayout.EndVertical();
		GUILayout.EndArea();
	}

	private void ScanCameraTimerOptionsIfNeeded(HandCamera cam)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)cam == (Object)null || _camTimerOptions.Count > 0)
		{
			return;
		}
		try
		{
			Type type = ((object)cam).GetType();
			List<int> list = new List<int>();
			FieldInfo[] fields = type.GetFields(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
			foreach (FieldInfo fieldInfo in fields)
			{
				if (fieldInfo.Name.ToLower().Contains("timer"))
				{
					if (fieldInfo.FieldType == typeof(int[]))
					{
						list.AddRange((int[])fieldInfo.GetValue(cam));
					}
					else if (fieldInfo.FieldType == typeof(List<int>))
					{
						list.AddRange((List<int>)fieldInfo.GetValue(cam));
					}
				}
			}
			PropertyInfo[] properties = type.GetProperties(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
			foreach (PropertyInfo propertyInfo in properties)
			{
				if (propertyInfo.Name.ToLower().Contains("timer"))
				{
					if (propertyInfo.PropertyType == typeof(int[]))
					{
						list.AddRange((int[])propertyInfo.GetValue(cam));
					}
					else if (propertyInfo.PropertyType == typeof(List<int>))
					{
						list.AddRange((List<int>)propertyInfo.GetValue(cam));
					}
				}
			}
			if (list.Count == 0)
			{
				list.AddRange(new int[8] { 0, 2, 3, 5, 8, 10, 15, 20 });
			}
			list = (from x in list.Distinct()
				where x >= 0 && x <= 120
				orderby x
				select x).ToList();
			_camTimerOptions.Clear();
			_camTimerOptions.AddRange(list);
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[CamUI] Failed to scan timer options: " + ex.Message);
			_camTimerOptions.Clear();
			_camTimerOptions.AddRange(new int[5] { 0, 2, 3, 5, 10 });
		}
	}

	private int TryGetCurrentTimerSeconds(HandCamera cam)
	{
		try
		{
			PropertyInfo propertyInfo = AccessTools.Property(((object)cam).GetType(), "CurrentTimer");
			if (propertyInfo != null && propertyInfo.PropertyType == typeof(int))
			{
				return (int)propertyInfo.GetValue(cam);
			}
			FieldInfo fieldInfo = AccessTools.Field(((object)cam).GetType(), "CurrentTimer");
			if (fieldInfo != null && fieldInfo.FieldType == typeof(int))
			{
				return (int)fieldInfo.GetValue(cam);
			}
			PropertyInfo propertyInfo2 = ((object)cam).GetType().GetProperties(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic).FirstOrDefault((PropertyInfo p) => p.PropertyType == typeof(int) && p.Name.ToLower().Contains("timer"));
			if (propertyInfo2 != null)
			{
				return (int)propertyInfo2.GetValue(cam);
			}
		}
		catch
		{
		}
		return 0;
	}

	private void SetCameraTimerToNearest(HandCamera cam, int seconds)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)cam == (Object)null)
		{
			return;
		}
		int num = seconds;
		if (_camTimerOptions.Count > 0)
		{
			int num2 = _camTimerOptions[0];
			int num3 = Math.Abs(num2 - seconds);
			for (int i = 1; i < _camTimerOptions.Count; i++)
			{
				int num4 = Math.Abs(_camTimerOptions[i] - seconds);
				if (num4 < num3)
				{
					num2 = _camTimerOptions[i];
					num3 = num4;
				}
			}
			num = num2;
		}
		bool flag = false;
		try
		{
			MethodInfo methodInfo = AccessTools.Method(((object)cam).GetType(), "SetTimer", new Type[1] { typeof(int) }, (Type[])null);
			if (methodInfo != null)
			{
				methodInfo.Invoke(cam, new object[1] { num });
				flag = true;
			}
		}
		catch
		{
		}
		if (!flag)
		{
			try
			{
				MethodInfo methodInfo2 = ((object)cam).GetType().GetMethods(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic).FirstOrDefault(delegate(MethodInfo m)
				{
					if (!m.Name.ToLower().Contains("timer"))
					{
						return false;
					}
					ParameterInfo[] parameters = m.GetParameters();
					return parameters.Length == 1 && parameters[0].ParameterType == typeof(int);
				});
				if (methodInfo2 != null)
				{
					methodInfo2.Invoke(cam, new object[1] { num });
					flag = true;
				}
			}
			catch
			{
			}
		}
		if (!flag)
		{
			try
			{
				PropertyInfo propertyInfo = AccessTools.Property(((object)cam).GetType(), "CurrentTimer");
				if (propertyInfo != null && propertyInfo.CanWrite && propertyInfo.PropertyType == typeof(int))
				{
					propertyInfo.SetValue(cam, num);
					flag = true;
				}
			}
			catch
			{
			}
		}
		if (!flag)
		{
			try
			{
				FieldInfo fieldInfo = AccessTools.Field(((object)cam).GetType(), "CurrentTimer");
				if (fieldInfo != null && fieldInfo.FieldType == typeof(int))
				{
					fieldInfo.SetValue(cam, num);
					flag = true;
				}
			}
			catch
			{
			}
		}
		if (flag)
		{
			_camTimerSec = num;
		}
		else
		{
			MelonLogger.Warning("[CamUI] Could not set camera timer via reflection.");
		}
	}

	private void SafeInvoke(Action a)
	{
		try
		{
			a?.Invoke();
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[CamUI] " + ex.Message);
		}
	}

	private void ComputeMapTransform()
	{
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_0213: Unknown result type (might be due to invalid IL or missing references)
		//IL_0218: Unknown result type (might be due to invalid IL or missing references)
		//IL_0224: Unknown result type (might be due to invalid IL or missing references)
		//IL_022b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0232: Unknown result type (might be due to invalid IL or missing references)
		//IL_023e: Unknown result type (might be due to invalid IL or missing references)
		//IL_024f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0256: Unknown result type (might be due to invalid IL or missing references)
		//IL_025d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0269: Unknown result type (might be due to invalid IL or missing references)
		float x = mapRefA.x;
		float y = mapRefA.y;
		float num = 1f;
		float x2 = mapRefB.x;
		float y2 = mapRefB.y;
		float num2 = 1f;
		float x3 = mapRefC.x;
		float y3 = mapRefC.y;
		float num3 = 1f;
		float num4 = x * (y2 * num3 - num2 * y3) - y * (x2 * num3 - num2 * x3) + num * (x2 * y3 - y2 * x3);
		if (Mathf.Abs(num4) < 1E-06f)
		{
			mapTransformReady = false;
			return;
		}
		float num5 = 1f / num4;
		float num6 = (y2 * num3 - num2 * y3) * num5;
		float num7 = (0f - (y * num3 - num * y3)) * num5;
		float num8 = (y * num2 - num * y2) * num5;
		float num9 = (0f - (x2 * num3 - num2 * x3)) * num5;
		float num10 = (x * num3 - num * x3) * num5;
		float num11 = (0f - (x * num2 - num * x2)) * num5;
		float num12 = (x2 * y3 - y2 * x3) * num5;
		float num13 = (0f - (x * y3 - y * x3)) * num5;
		float num14 = (x * y2 - y * x2) * num5;
		float x4 = worldRefA.x;
		float x5 = worldRefB.x;
		float x6 = worldRefC.x;
		float y4 = worldRefA.y;
		float y5 = worldRefB.y;
		float y6 = worldRefC.y;
		Vector3 val = default(Vector3);
		((Vector3)(ref val))._002Ector(num6 * x4 + num7 * x5 + num8 * x6, num9 * x4 + num10 * x5 + num11 * x6, num12 * x4 + num13 * x5 + num14 * x6);
		Vector3 val2 = default(Vector3);
		((Vector3)(ref val2))._002Ector(num6 * y4 + num7 * y5 + num8 * y6, num9 * y4 + num10 * y5 + num11 * y6, num12 * y4 + num13 * y5 + num14 * y6);
		mapTransform = Matrix4x4.identity;
		((Matrix4x4)(ref mapTransform)).SetRow(0, new Vector4(val.x, val.y, val.z, 0f));
		((Matrix4x4)(ref mapTransform)).SetRow(1, new Vector4(val2.x, val2.y, val2.z, 0f));
		mapTransformReady = true;
		MelonLogger.Msg("[MiniMap] Map transform computed (affine from 3 refs).");
	}

	private Vector3 MapToWorld3Point(Vector3 localClick)
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0043: Unknown result type (might be due to invalid IL or missing references)
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_0055: Unknown result type (might be due to invalid IL or missing references)
		Vector3 val = default(Vector3);
		((Vector3)(ref val))._002Ector(localClick.x, localClick.y, 1f);
		float num = Vector3.Dot(Vector4.op_Implicit(((Matrix4x4)(ref mapTransform)).GetRow(0)), val);
		float num2 = Vector3.Dot(Vector4.op_Implicit(((Matrix4x4)(ref mapTransform)).GetRow(1)), val);
		return new Vector3(num, 300f, num2);
	}

	private void TeleportToHighestPoint(Vector3 worldPos)
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0008: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Unknown result type (might be due to invalid IL or missing references)
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		RaycastHit val = default(RaycastHit);
		if (Physics.Raycast(new Ray(new Vector3(worldPos.x, 2000f, worldPos.z), Vector3.down), ref val, 5000f, -1, (QueryTriggerInteraction)1))
		{
			Vector3 val2 = ((RaycastHit)(ref val)).point + Vector3.up * 1f;
			TeleportLocalPlayer(val2, (LocomotionFunction)0);
			MelonLogger.Msg($"[MiniMap] Teleported to highest point at {val2}");
		}
		else
		{
			TeleportLocalPlayer(new Vector3(worldPos.x, 300f, worldPos.z), (LocomotionFunction)0);
			MelonLogger.Msg("[MiniMap] No surface found; used fallback height.");
		}
	}

	private void ToggleMiniMap()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		//IL_02e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ec: Expected O, but got Unknown
		//IL_0525: Unknown result type (might be due to invalid IL or missing references)
		//IL_04ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_0502: Unknown result type (might be due to invalid IL or missing references)
		//IL_0335: Unknown result type (might be due to invalid IL or missing references)
		//IL_034a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0370: Unknown result type (might be due to invalid IL or missing references)
		//IL_0389: Unknown result type (might be due to invalid IL or missing references)
		//IL_0394: Expected O, but got Unknown
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Expected O, but got Unknown
		//IL_03a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_03a7: Expected O, but got Unknown
		//IL_03b4: Unknown result type (might be due to invalid IL or missing references)
		//IL_03b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Expected O, but got Unknown
		//IL_00ae: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Expected O, but got Unknown
		//IL_00c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0136: Expected O, but got Unknown
		//IL_013e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0148: Expected O, but got Unknown
		//IL_0163: Unknown result type (might be due to invalid IL or missing references)
		//IL_016e: Expected O, but got Unknown
		//IL_01e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f3: Expected O, but got Unknown
		//IL_01f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_0202: Expected O, but got Unknown
		//IL_0424: Unknown result type (might be due to invalid IL or missing references)
		//IL_042b: Expected O, but got Unknown
		//IL_019c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a6: Expected O, but got Unknown
		//IL_0282: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)boardClone == (Object)null)
		{
			PlayerController current = PlayerController.Current;
			Camera val = (((Object)current != (Object)null) ? current.Camera : null);
			if ((Object)val == (Object)null)
			{
				return;
			}
			GameObject val2 = ((IEnumerable<GameObject>)Resources.FindObjectsOfTypeAll<GameObject>()).FirstOrDefault((Func<GameObject, bool>)((GameObject val7) => (Object)val7 != (Object)null && ((Object)val7).name == "Map Board"));
			if ((Object)val2 == (Object)null)
			{
				MelonLogger.Warning("[MiniMap] Map Board prefab not found.");
				return;
			}
			boardClone = Object.Instantiate<GameObject>(val2, ((Component)val).transform);
			((Object)boardClone).name = "Minimap";
			Object.DontDestroyOnLoad((Object)boardClone);
			boardClone.transform.localPosition = minimapOffset;
			boardClone.transform.localRotation = Quaternion.identity;
			boardClone.transform.localScale = minimapScale;
			Transform val3 = ((IEnumerable<Transform>)boardClone.GetComponentsInChildren<Transform>(true)).FirstOrDefault((Func<Transform, bool>)((Transform t) => ((Object)t).name.IndexOf("Hand Touch", StringComparison.OrdinalIgnoreCase) >= 0));
			if ((Object)val3 != (Object)null)
			{
				Object.Destroy((Object)((Component)val3).gameObject);
			}
			Component[] components = boardClone.GetComponents<Component>();
			foreach (Component val4 in components)
			{
				if (!((Object)val4 == (Object)null))
				{
					string name = ((object)val4).GetType().Name;
					if (name == "NetworkEntity" || name == "NetworkPrefab")
					{
						Object.DestroyImmediate((Object)val4);
					}
				}
			}
			MapBoard component = boardClone.GetComponent<MapBoard>();
			MapLandmarkDisplay component2 = boardClone.GetComponent<MapLandmarkDisplay>();
			MethodInfo methodInfo = AccessTools.Method(typeof(MapLandmarkDisplay), "Setup", (Type[])null, (Type[])null);
			if ((Object)component == (Object)null || (Object)component2 == (Object)null || methodInfo == null)
			{
				MelonLogger.Warning("[MiniMap] Setup failed (missing MapBoard / MapLandmarkDisplay).");
				return;
			}
			AccessTools.Field(typeof(MapBoard), "zoom")?.SetValue(component, 1f);
			AccessTools.Field(typeof(MapBoard), "syncBoard")?.SetValue(component, null);
			AccessTools.Field(typeof(MapBoard), "positionOffset")?.SetValue(component, Vector2.zero);
			try
			{
				AccessTools.Method(typeof(MapBoard), "SetupAsClient", (Type[])null, (Type[])null)?.Invoke(component, null);
			}
			catch
			{
				MelonLogger.Warning("[MiniMap] SetupAsClient failed.");
			}
			methodInfo.Invoke(component2, new object[1] { component });
			if ((Object)minimapOutline == (Object)null)
			{
				minimapOutline = GameObject.CreatePrimitive((PrimitiveType)5);
				((Object)minimapOutline).name = "Minimap Outline";
				minimapOutline.transform.SetParent(boardClone.transform, false);
				minimapOutline.transform.localPosition = outlineOffset;
				minimapOutline.transform.localRotation = Quaternion.identity;
				minimapOutline.transform.localScale = new Vector3(outlineWidth, outlineHeight, 1f);
				Renderer component3 = minimapOutline.GetComponent<Renderer>();
				if ((Object)component3 != (Object)null)
				{
					Material val5 = new Material(Shader.Find("Unlit/Color"));
					val5.color = Color32.op_Implicit(new Color32((byte)33, (byte)33, (byte)33, byte.MaxValue));
					component3.material = val5;
				}
			}
			try
			{
				if (AccessTools.Field(typeof(MapLandmarkDisplay), "icons")?.GetValue(component2) is IDictionary dictionary)
				{
					foreach (object value in dictionary.Values)
					{
						LandmarkIcon val6 = (LandmarkIcon)((value is LandmarkIcon) ? value : null);
						if ((Object)(object)val6 != (Object)null && ((Object)val6).name == "Other Player Icon")
						{
							AccessTools.Field(typeof(LandmarkIcon), "scale")?.SetValue(val6, 0.04f);
							MelonLogger.Msg("[MiniMap] Set 'Other Player Icon' scale to 0.04f.");
						}
					}
				}
			}
			catch (Exception ex)
			{
				MelonLogger.Warning("[MiniMap] Failed to update icon scale: " + ex.Message);
			}
			ComputeMapTransform();
			MelonLogger.Msg("[MiniMap] Minimap spawned, finalized, and frozen.");
		}
		minimapVisible = !minimapVisible;
		if (minimapVisible)
		{
			boardClone.transform.localPosition = minimapOffset;
			boardClone.transform.localScale = minimapScale;
			boardClone.SetActive(true);
		}
		else
		{
			boardClone.transform.localPosition = boardHiddenPos;
			boardClone.SetActive(false);
		}
	}

	private void Minimap_Tick()
	{
		TeleportFromMinimapClick();
	}

	private void TeleportFromMinimapClick()
	{
		//IL_000e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Expected O, but got Unknown
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_0052: Expected O, but got Unknown
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Expected O, but got Unknown
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		//IL_0091: Unknown result type (might be due to invalid IL or missing references)
		//IL_0096: Unknown result type (might be due to invalid IL or missing references)
		//IL_009b: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c2: Expected O, but got Unknown
		//IL_00ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_0101: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0119: Unknown result type (might be due to invalid IL or missing references)
		if (!minimapVisible || (Object)boardClone == (Object)null || !boardClone.activeInHierarchy)
		{
			return;
		}
		Mouse current = Mouse.current;
		if (current == null || !current.leftButton.wasPressedThisFrame)
		{
			return;
		}
		PlayerController current2 = PlayerController.Current;
		Camera val = (((Object)current2 != (Object)null) ? current2.Camera : null);
		if (!((Object)val == (Object)null))
		{
			RaycastHit val2 = default(RaycastHit);
			if (!mapTransformReady)
			{
				MelonLogger.Warning("[MiniMap] Click ignored: map transform not ready (need 3 ref pairs).");
			}
			else if (Physics.Raycast(val.ScreenPointToRay(Vector2.op_Implicit(((InputControl<Vector2>)(object)((Pointer)current).position).ReadValue())), ref val2, 200f, -1, (QueryTriggerInteraction)2) && (Object)((RaycastHit)(ref val2)).transform != (Object)null && ((RaycastHit)(ref val2)).transform.IsChildOf(boardClone.transform))
			{
				Vector3 localClick = boardClone.transform.InverseTransformPoint(((RaycastHit)(ref val2)).point);
				Vector3 val3 = MapToWorld3Point(localClick);
				TeleportToHighestPoint(val3);
				MelonLogger.Msg($"[MiniMap] Teleported near {val3.x:F1}, {val3.z:F1}");
			}
		}
	}

	private void PFV3_DrawStakeDrawSection()
	{
		GUILayout.Space(10f);
		GUILayout.Label("Drawing", _h2, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Toggle(PancakesBehaviour.drawModeActive, PancakesBehaviour.drawModeActive ? "Drawing… (ESC to exit)" : "Enable", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }) != PancakesBehaviour.drawModeActive)
		{
			PFV3_ToggleStakeDraw();
		}
		GUILayout.FlexibleSpace();
		GUILayout.Label($"Spacing: {PancakesBehaviour.drawSpacing:0.00}m", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
		float num = GUILayout.HorizontalSlider(PancakesBehaviour.drawSpacing, 0.05f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(160f) });
		if (Mathf.Abs(num - PancakesBehaviour.drawSpacing) > 0.0001f)
		{
			PancakesBehaviour.drawSpacing = num;
		}
		GUILayout.EndHorizontal();
		GUILayout.Space(2f);
	}

	private void PFV3_ToggleStakeDraw()
	{
		//IL_004d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Expected O, but got Unknown
		if (PancakesBehaviour.drawModeActive)
		{
			try
			{
				if (!_showUI)
				{
					_showUI = true;
				}
			}
			catch
			{
			}
			PFV3_SetEditorCamera(null);
			PFV3_EnsureBodyCamActive();
			PancakesBehaviour.ToggleDrawModeOff();
			return;
		}
		PancakesBehaviour.ToggleDrawModeOn(this);
		try
		{
			if (_showUI)
			{
				_showUI = false;
			}
		}
		catch
		{
		}
		if ((Object)PancakesBehaviour.drawCam != (Object)null)
		{
			PFV3_SetEditorCamera(PancakesBehaviour.drawCam);
		}
	}

	private void PFV3_SetEditorCamera(Camera camOrNull)
	{
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_0052: Expected O, but got Unknown
		try
		{
			FieldInfo fieldInfo = AccessTools.Field(typeof(AdonaiUnifiedMod), "_pfv3EditorCam");
			if (fieldInfo != null)
			{
				fieldInfo.SetValue(this, camOrNull);
			}
			FieldInfo fieldInfo2 = AccessTools.Field(typeof(AdonaiUnifiedMod), "_pfv3EditorOn");
			if (fieldInfo2 != null)
			{
				fieldInfo2.SetValue(this, (Object)camOrNull != (Object)null);
			}
			FieldInfo fieldInfo3 = AccessTools.Field(typeof(AdonaiUnifiedMod), "_pfv3ActiveCamera");
			if (fieldInfo3 != null)
			{
				fieldInfo3.SetValue(this, camOrNull);
			}
		}
		catch
		{
		}
	}

	private void PFV3_EnsureBodyCamActive()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Expected O, but got Unknown
		try
		{
			PlayerController current = PlayerController.Current;
			if ((Object)current != (Object)null && (Object)current.Camera != (Object)null)
			{
				((Behaviour)current.Camera).enabled = true;
			}
		}
		catch
		{
		}
	}

	private void DrawSpawnAnythingCard()
	{
		DrawCard("Spawn Anything", delegate
		{
			GUILayout.Label("Runs the built-in spawn console commands. Prefab name can be partial (use multi for mass).", _small, Array.Empty<GUILayoutOption>());
			_spawnPrefab = GUILayout.TextField(_spawnPrefab ?? string.Empty, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			_spawnArgs = GUILayout.TextField(_spawnArgs ?? string.Empty, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(22f) });
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Spawn (server)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				TrySendGameCommand(BuildSpawnCommand("spawn"));
			}
			if (GUILayout.Button("Spawn Local", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				TrySendGameCommand(BuildSpawnCommand("spawn local"));
			}
			GUILayout.EndHorizontal();
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Spawn Multi (matches)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				TrySendGameCommand(BuildSpawnCommand("spawn multi"));
			}
			if (GUILayout.Button("Drop Package/Collection", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				TrySendGameCommand(BuildSpawnCommand("spawn drop"));
			}
			GUILayout.EndHorizontal();
		});
	}

	private string BuildSpawnCommand(string prefix)
	{
		List<string> list = new List<string> { prefix };
		if (!string.IsNullOrWhiteSpace(_spawnPrefab))
		{
			list.Add(_spawnPrefab.Trim());
		}
		if (!string.IsNullOrWhiteSpace(_spawnArgs))
		{
			list.Add(_spawnArgs.Trim());
		}
		return string.Join(" ", list.Where((string p) => !string.IsNullOrWhiteSpace(p)));
	}

	private void DrawProgressionCard()
	{
		DrawCard("Progression / QoL Cheats", delegate
		{
			GUILayout.Label("Server-side progress helpers from the built-in 'progress' module.", _small, Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Finish all repair boxes", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("progress finishboxes");
			}
			if (GUILayout.Button("Fill all books", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("progress fill-books");
			}
			if (GUILayout.Button("Fill community boxes", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("progress fillcommunityboxes");
			}
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label("Cave depth:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			_progressCaveDepth = GUILayout.TextField(_progressCaveDepth ?? "6", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
			if (GUILayout.Button("Generate caves", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(130f) }))
			{
				TrySendGameCommand("progress generatecaves " + _progressCaveDepth);
			}
			if (GUILayout.Button("Repair cave teleporters", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(170f) }))
			{
				TrySendGameCommand("progress repaircaveteleporters " + _progressCaveDepth);
			}
			GUILayout.EndHorizontal();
		});
	}

	private void DrawWorldPacingCard()
	{
		DrawCard("World Pacing / Time", delegate
		{
			GUILayout.Label("Hooks into the 'time' console module.", _small, Array.Empty<GUILayoutOption>());
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Day (12:00)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("time set 12:00");
			}
			if (GUILayout.Button("Night (00:00)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("time set 00:00");
			}
			if (GUILayout.Button("Dusk (18:00)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("time set 18:00");
			}
			GUILayout.EndHorizontal();
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label("Set time (hh:mm):", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(120f) });
			_timeSetInput = GUILayout.TextField(_timeSetInput ?? "12:00", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			if (GUILayout.Button("Apply", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
			{
				TrySendGameCommand("time set " + _timeSetInput);
			}
			GUILayout.EndHorizontal();
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label("Advance minutes:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(120f) });
			_timeFutureMinutes = GUILayout.TextField(_timeFutureMinutes ?? "60", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			if (GUILayout.Button("Advance", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) }))
			{
				TrySendGameCommand("time future " + _timeFutureMinutes);
			}
			if (GUILayout.Button("Toggle Day/Night", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(140f) }))
			{
				TrySendGameCommand("time toggle");
			}
			GUILayout.EndHorizontal();
		});
	}

	private void DrawServerSettingsCard()
	{
		DrawCard("Server / Settings Access", delegate
		{
			GUILayout.Label("Talks to the built-in 'settings' commands. Requires server permission.", _small, Array.Empty<GUILayoutOption>());
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Enable settings board", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("settings enable-board");
			}
			if (GUILayout.Button("Disable settings board", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("settings disable-board");
			}
			GUILayout.EndHorizontal();
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label("Heat multiplier", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(110f) });
			_heatMultiplierText = GUILayout.TextField(_heatMultiplierText ?? "1.0", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			if (GUILayout.Button("Apply", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
			{
				TrySendGameCommand("settings heat " + _heatMultiplierText);
			}
			GUILayout.EndHorizontal();
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label("Toggle setting", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(110f) });
			_settingsToggleName = GUILayout.TextField(_settingsToggleName ?? "", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(200f) });
			if (GUILayout.Button("Toggle", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
			{
				TrySendGameCommand("settings toggle " + _settingsToggleName);
			}
			GUILayout.EndHorizontal();
			GUILayout.Label("Change setting (target setting value):", _small, Array.Empty<GUILayoutOption>());
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			_settingsTarget = GUILayout.TextField(_settingsTarget ?? "game", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			_settingsName = GUILayout.TextField(_settingsName ?? "", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(160f) });
			_settingsValue = GUILayout.TextField(_settingsValue ?? "", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(100f) });
			if (GUILayout.Button("Apply", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
			{
				TrySendGameCommand("settings changesetting " + _settingsTarget + " " + _settingsName + " " + _settingsValue);
			}
			GUILayout.EndHorizontal();
			if (GUILayout.Button("Save settings", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
			{
				TrySendGameCommand("settings save");
			}
		});
	}

	private bool TrySendGameCommand(string command)
	{
		if (string.IsNullOrWhiteSpace(command))
		{
			return false;
		}
		command = command.Trim();
		try
		{
			Type? type = Type.GetType("ATT.Character.QuickAccessMenu.CommandSync");
			object obj = (type?.GetField("Instance", BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic))?.GetValue(null);
			MethodInfo methodInfo = type?.GetMethod("RouteCommand", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, new Type[2]
			{
				typeof(string),
				typeof(Action<string, string>)
			}, null);
			if (obj != null && methodInfo != null)
			{
				methodInfo.Invoke(obj, new object[2] { command, null });
				_statusLeft = "Sent: " + command;
				return true;
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified/Commands] CommandSync route failed: " + ex.GetBaseException().Message);
		}
		try
		{
			MethodInfo methodInfo2 = Type.GetType("Alta.Console.CommandService")?.GetMethod("Handle", BindingFlags.Static | BindingFlags.Public);
			if (methodInfo2 != null)
			{
				if (methodInfo2.Invoke(null, new object[2] { command, null }) is Task { IsCompleted: false } task)
				{
					task.ConfigureAwait(continueOnCapturedContext: false);
				}
				_statusLeft = "Executed locally: " + command;
				return true;
			}
		}
		catch (Exception ex2)
		{
			MelonLogger.Warning("[Unified/Commands] CommandService handle failed: " + ex2.GetBaseException().Message);
		}
		_statusLeft = "Could not send command (CommandSync missing).";
		return false;
	}

	private void ToggleUnifiedMenu()
	{
		SetUnifiedMenuVisible(!_menuVisible);
	}

	private void SetUnifiedMenuVisible(bool show)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		_menuVisible = show;
		if (!show)
		{
			_menuBounds = Rect.zero;
		}
	}

	private void HandleMenuShortcuts()
	{
		Keyboard current = Keyboard.current;
		if (current != null && ((ButtonControl)current[(Key)94]).wasPressedThisFrame)
		{
			ToggleUnifiedMenu();
		}
		else if (_menuVisible && current != null && ((ButtonControl)current.escapeKey).wasPressedThisFrame)
		{
			SetUnifiedMenuVisible(show: false);
		}
	}

	private void DrawUnifiedMenu()
	{
		//IL_0061: Unknown result type (might be due to invalid IL or missing references)
		//IL_0073: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Expected O, but got Unknown
		//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_0104: Unknown result type (might be due to invalid IL or missing references)
		//IL_010e: Expected O, but got Unknown
		if (_menuVisible)
		{
			float num = Mathf.Clamp((float)Screen.width * 0.62f, 560f, 860f);
			float num2 = Mathf.Clamp((float)Screen.height * 0.72f, 420f, (float)Screen.height - 60f);
			float num3 = ((float)Screen.width - num) * 0.5f;
			float num4 = 24f;
			Rect val = default(Rect);
			((Rect)(ref val))._002Ector(num3, num4, num, num2);
			_menuBounds = val;
			GUILayout.BeginArea(val, _card);
			GUILayout.BeginVertical(Array.Empty<GUILayoutOption>());
			DrawMenuHeader();
			DrawMenuTabs();
			GUILayout.Space(6f);
			GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 2f), (Texture)_texDivider);
			Vector2 scrollForTab = GetScrollForTab();
			scrollForTab = GUILayout.BeginScrollView(scrollForTab, Array.Empty<GUILayoutOption>());
			DrawCurrentTab();
			GUILayout.EndScrollView();
			StoreScrollForTab(scrollForTab);
			GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 2f), (Texture)_texDivider);
			GUILayout.Label(GetMenuFooter(), _small, Array.Empty<GUILayoutOption>());
			GUILayout.EndVertical();
			GUILayout.EndArea();
		}
	}

	private void DrawMenuHeader()
	{
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Unified Controls", _h1, Array.Empty<GUILayoutOption>());
		GUILayout.FlexibleSpace();
		GUILayout.Label(_sceneReady ? "In world" : "Not in world", _small, Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Close", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
		{
			SetUnifiedMenuVisible(show: false);
		}
		GUILayout.EndHorizontal();
	}

	private void DrawMenuTabs()
	{
		bool sceneReady = _sceneReady;
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Toggle(_menuTab == MenuTab.Connect, "Connect", _topTab, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
		{
			_menuTab = MenuTab.Connect;
		}
		bool enabled = GUI.enabled;
		GUI.enabled = enabled && sceneReady;
		if (GUILayout.Toggle(_menuTab == MenuTab.General, "General", _topTab, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
		{
			_menuTab = MenuTab.General;
		}
		if (GUILayout.Toggle(_menuTab == MenuTab.Utilities, "Utilities", _topTab, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
		{
			_menuTab = MenuTab.Utilities;
		}
		GUI.enabled = enabled;
		GUILayout.FlexibleSpace();
		GUILayout.EndHorizontal();
		if (!sceneReady && _menuTab != MenuTab.Connect)
		{
			_menuTab = MenuTab.Connect;
		}
	}
private void DrawUtilitiesTab()
{
    // Optional: make it scrollable if you add many utility cards
    _dropScroll = GUILayout.BeginScrollView(_dropScroll, false, true, GUILayout.Height(400));

    // Call your spawn menu
    DrawSpawnAnythingCard();

    GUILayout.EndScrollView();
}
	private Vector2 GetScrollForTab()
	{
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		return (Vector2)(_menuTab switch
		{
			MenuTab.General => _menuScrollGeneral, 
			MenuTab.Utilities => _menuScrollUtilities, 
			_ => _menuScrollConnect, 
		});
	}

	private void StoreScrollForTab(Vector2 scroll)
	{
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		switch (_menuTab)
		{
		case MenuTab.General:
			_menuScrollGeneral = scroll;
			break;
		case MenuTab.Utilities:
			_menuScrollUtilities = scroll;
			break;
		default:
			_menuScrollConnect = scroll;
			break;
		}
	}

private void DrawCurrentTab()
{
    switch (_menuTab)
    {
        case MenuTab.General:
            DrawGeneralTabUi();
            break;

        case MenuTab.Utilities:
            // Draw the Utilities tab and include the spawn menu
            DrawUtilitiesTabUi();
            break;

        default:
            DrawConnectTabUi();
            break;
    }
}


	private void DrawConnectTabUi()
	{
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		DrawCard("Allowed Servers", delegate
		{
			if (AllowedServers.Count == 0)
			{
				GUILayout.Label("No allowed servers configured.", _small, Array.Empty<GUILayoutOption>());
				return;
			}
			foreach (KeyValuePair<int, string> allowedServer in AllowedServers)
			{
				if (GUILayout.Button($"{allowedServer.Value} ({allowedServer.Key})", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
				{
					if (!IsServerAllowed(allowedServer.Key))
					{
						QuitForDisallowed("Blocked UI join attempt: " + allowedServer.Key);
					}
					else
					{
						JoinAllowedServer(allowedServer.Key);
					}
				}
			}
		});
		if (!string.IsNullOrEmpty(_connectErr))
		{
			GUI.contentColor = Color.red;
			GUILayout.Label(_connectErr, _small, Array.Empty<GUILayoutOption>());
			GUI.contentColor = Color.white;
		}
		if (GUILayout.Button("Quit Game", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
		{
			Application.Quit();
		}
		DrawCard("API Status", delegate
		{
			GUILayout.Label(GetApiStatusText(), _small, Array.Empty<GUILayoutOption>());
			if (_awaitingToken)
			{
				if (GUILayout.Button("Focus Chat Input", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					RequestChatFocus();
				}
			}
			else if (GUILayout.Button("Clear Saved Token", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				ClearApiToken();
			}
		});
	}

	private string GetApiStatusText()
	{
		if (_awaitingToken)
		{
			return "Paste your API token into the chat box and press Enter. It will be saved for future sessions.";
		}
		return "Authenticated with the Basemodal API. Incoming chat will appear in the overlay.";
	}

	private void SaveShortcutFromUi()
	{
		if (!string.IsNullOrWhiteSpace(_srvName) && !string.IsNullOrWhiteSpace(_srvId))
		{
			_saved.Add(new ServerEntry
			{
				Name = _srvName,
				Id = _srvId
			});
			_selServer = _saved.Count - 1;
			_srvName = string.Empty;
			SavePersisted();
		}
	}

	private void DrawGeneralTabUi()
	{
		DrawCard("Powers & Visuals", delegate
		{
			DrawToggle("Dev Light", Actions.is_devlight, delegate(bool v)
			{
				if (v != Actions.is_devlight)
				{
					Actions.toggle_devlight();
				}
			});
			DrawToggle("Blackhole", _bhEnabled, BHSetEnabled);
			DrawToggle("Cruel Sun", _csEnabled, CSSetEnabled);
			DrawToggle("Smite", _smiteEnabled, SmiteSetEnabled);
			DrawToggle("Blade Held FX", _bladeHeldFxEnabled, BladeHeldFxSetEnabled);
			DrawToggle("Spinning", _isSpinning, delegate
			{
				ToggleSpinning();
			});
		});
		DrawCard("Teleport Particle Shapes", delegate
		{
			DrawToggle("Sphere", _teleportSphereEnabled, SetTeleportSphereEnabled);
			DrawToggle("Cone", _teleportConeEnabled, SetTeleportConeEnabled);
			DrawToggle("Star", _teleportStarEnabled, SetTeleportStarEnabled);
			DrawToggle("Ring", _teleportRingEnabled, SetTeleportRingEnabled);
			DrawToggle("Ring Arena", _teleportRingArenaEnabled, SetTeleportRingArenaEnabled);
			DrawToggle("Cube", _teleportCubeEnabled, SetTeleportCubeEnabled);
			DrawToggle("Helix", _teleportHelixEnabled, SetTeleportHelixEnabled);
			GUILayout.Label($"Shape Scale: {_teleportShapeScale:0.00}x", _h3, Array.Empty<GUILayoutOption>());
			float num = GUILayout.HorizontalSlider(_teleportShapeScale, 0.2f, 3f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			if (Mathf.Abs(num - _teleportShapeScale) > 0.0005f)
			{
				_teleportShapeScale = num;
			}
		});
		DrawCard("Body", delegate
		{
			DrawBodyBendMultiplierSlider();
		});
		DrawCard("ESP", delegate
		{
			//IL_0399: Unknown result type (might be due to invalid IL or missing references)
			//IL_03c5: Unknown result type (might be due to invalid IL or missing references)
			DrawToggle("Enable ESP", espEnabled, delegate(bool v)
			{
				if (v)
				{
					StartESP();
				}
				else
				{
					StopESP();
				}
			});
			GUILayout.Label("Filter (name contains):", _h3, Array.Empty<GUILayoutOption>());
			espInput = GUILayout.TextField(espInput, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			if (GUILayout.Button("Clear ESP Markers", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
			{
				ClearAllESPMarkers();
			}
			GUILayout.Label($"Refresh Interval: {espRefreshInterval:F1}s", _h3, Array.Empty<GUILayoutOption>());
			espRefreshInterval = GUILayout.HorizontalSlider(espRefreshInterval, 0.1f, 10f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			DrawToggle("Show Labels", espShowLabels, delegate(bool v)
			{
				espShowLabels = v;
			});
			DrawToggle("Parent to Target", espParentToTarget, delegate(bool v)
			{
				espParentToTarget = v;
			});
			DrawToggle("Copy Rotation", espCopyRotation, delegate(bool v)
			{
				espCopyRotation = v;
			});
			DrawToggle("One Per Root", espOnePerRoot, delegate(bool v)
			{
				espOnePerRoot = v;
			});
			GUILayout.Label($"World Size: {espWorldSize:F2}", _h3, Array.Empty<GUILayoutOption>());
			espWorldSize = GUILayout.HorizontalSlider(espWorldSize, 0.05f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			GUILayout.Label($"Label Max Length: {espLabelMaxLen}", _h3, Array.Empty<GUILayoutOption>());
			espLabelMaxLen = Mathf.RoundToInt(GUILayout.HorizontalSlider((float)espLabelMaxLen, 5f, 50f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }));
			GUILayout.Label($"Label Y Offset: {espLabelYOffset:F2}", _h3, Array.Empty<GUILayoutOption>());
			espLabelYOffset = GUILayout.HorizontalSlider(espLabelYOffset, 0f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			GUILayout.Label($"Label Base Size: {espLabelBaseSize:F2}", _h3, Array.Empty<GUILayoutOption>());
			espLabelBaseSize = GUILayout.HorizontalSlider(espLabelBaseSize, 0.1f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			GUILayout.Label("ESP Color:", _h3, Array.Empty<GUILayoutOption>());
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			espColor.r = GUILayout.HorizontalSlider(espColor.r, 0f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			espColor.g = GUILayout.HorizontalSlider(espColor.g, 0f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			espColor.b = GUILayout.HorizontalSlider(espColor.b, 0f, 1f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
			GUILayout.EndHorizontal();
			GUI.color = espColor;
			GUILayout.Label("■", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(20f) });
			GUI.color = Color.white;
		});
		DrawCard("Safety Suite", delegate
		{
			DrawToggle("Invis", _invisEnabled, SetInvisEnabled);
			DrawToggle("Void Guard", Actions.no_void_tp, delegate(bool v)
			{
				Actions.no_void_tp = v;
			});
			DrawToggle("Mini Map", minimapVisible, SetMinimapVisible);
			DrawToggle("Third Person", thirdpersonMode, delegate
			{
				ToggleThirdPersonCamera();
			});
			bool enabled = GUI.enabled;
			GUI.enabled = enabled && minimapVisible;
			DrawToggle("Follow Camera", _mapFollowEnabled, delegate(bool v)
			{
				_mapFollowEnabled = v;
			});
			if (GUILayout.Button(GetFollowModeLabel(), _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
			{
				ToggleFollowMode();
			}
			GUI.enabled = enabled;
		});
		DrawCard("Teleport & Control", delegate
		{
			//IL_0068: Unknown result type (might be due to invalid IL or missing references)
			if (_teleports.Count == 0)
			{
				GUILayout.Label("No teleport locations available.", _small, Array.Empty<GUILayoutOption>());
			}
			else
			{
				foreach (KeyValuePair<string, Vector3> teleport in _teleports)
				{
					if (GUILayout.Button(teleport.Key, _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
					{
						TeleportLocalPlayer(teleport.Value, (LocomotionFunction)0);
					}
				}
			}
			GUILayout.Space(4f);
			GUILayout.Label("Custom Coords (X,Y,Z):", _h3, Array.Empty<GUILayoutOption>());
			_customTeleportInput = GUILayout.TextField(_customTeleportInput, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
			if (GUILayout.Button("Teleport to Coords", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
			{
				TryTeleportToCustomCoords();
			}
			GUILayout.Space(4f);
			if (GUILayout.Button(_controlActive ? "Exit Control" : "Take Control", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
			{
				SetControlActive(!_controlActive);
			}
			if (GUILayout.Button("Disconnect", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) }))
			{
				DisconnectToMenu();
			}
		});
	}

	private void DrawUtilitiesTabUi()
	{
		DrawCard("Utilities", delegate
		{
			//IL_0005: Unknown result type (might be due to invalid IL or missing references)
			//IL_0010: Expected O, but got Unknown
			//IL_024a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0255: Expected O, but got Unknown
			//IL_025e: Unknown result type (might be due to invalid IL or missing references)
			//IL_02b9: Unknown result type (might be due to invalid IL or missing references)
			//IL_02c4: Expected O, but got Unknown
			//IL_02c8: Unknown result type (might be due to invalid IL or missing references)
			//IL_02d3: Expected O, but got Unknown
			//IL_02de: Unknown result type (might be due to invalid IL or missing references)
			//IL_02e3: Unknown result type (might be due to invalid IL or missing references)
			//IL_02ed: Unknown result type (might be due to invalid IL or missing references)
			//IL_02f2: Unknown result type (might be due to invalid IL or missing references)
			if ((Object)PlayerController.Current != (Object)null && ((Object)PlayerController.Current).name == "Customization Controller(Clone)" && GUILayout.Button("Escape Customization", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				try
				{
					Actions.escape_customization();
				}
				catch (Exception ex)
				{
					MelonLogger.Warning("escape_customization failed: " + ex.Message);
				}
			}
			DrawToggle("Auto Tree Harvest", _autoHarvestEnabled, SetAutoHarvestEnabled);
			DrawToggle("Auto Revive", _autoReviveEnabled, SetAutoReviveEnabled);
			if (GUILayout.Button("Revive Me Now", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				try
				{
					Actions.revive();
				}
				catch (Exception ex2)
				{
					MelonLogger.Warning("revive failed: " + ex2.Message);
				}
			}
			if (GUILayout.Button("Toggle Enemy Health Bars", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				ToggleEnemyHealthBars val = ScriptableObject.CreateInstance<ToggleEnemyHealthBars>();
				if ((Object)(object)val != (Object)null)
				{
					((QuickAccessMenuAction)val).Run((Controller)null);
				}
			}
			if (GUILayout.Button("Equipment Mode", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				EquipmentModeQuickAccess val2 = ScriptableObject.CreateInstance<EquipmentModeQuickAccess>();
				if ((Object)(object)val2 != (Object)null)
				{
					((QuickAccessMenuAction)val2).Run((Controller)null);
				}
			}
			if (GUILayout.Button("Summon Camera", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				SummonCameraQuickAccess val3 = ScriptableObject.CreateInstance<SummonCameraQuickAccess>();
				if ((Object)(object)val3 != (Object)null)
				{
					((QuickAccessMenuAction)val3).Run((Controller)null);
				}
			}
			if (GUILayout.Button("Toggle Terrain", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				ToggleTerrain();
			}
			if (GUILayout.Button("Bag Looking", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				ToggleStorage();
			}
			if (GUILayout.Button("Teleport Player -> Camera", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				Camera main = Camera.main;
				if ((Object)main != (Object)null)
				{
					TeleportLocalPlayer(((Component)main).transform.position, (LocomotionFunction)0);
				}
			}
			if (GUILayout.Button("Teleport Camera -> Player", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{
				PlayerController current = PlayerController.Current;
				Camera main2 = Camera.main;
				Transform val4 = (((Object)(object)main2 != (Object)null) ? ((Component)main2).transform : null);
				if ((Object)current != (Object)null && (Object)val4 != (Object)null)
				{
					val4.position = ((Component)current).transform.position + Vector3.up * 1.6f;
				}
			}
		});
		DrawCard("Debug / Permissions", delegate
		{
			DrawToggle("Force Debug Features", ForceDebugPermissions, SetDebugPermissionBypass);
			DrawToggle("Impact Hit Debug", _impactDebugSubscribed, SetImpactDebugSubscribed);
			DrawToggle("Damage Debug Overlay", _damageDebugEnabled, SetDamageDebugEnabled);
			DrawToggle("Mod Mic Mute (self)", _modMicMuteActive, SetModMicMute);
		});
		DrawCard("Teleport Attach Offset", delegate
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Expected O, but got Unknown
			CSRuntime instance = CSRuntime.Instance;
			if ((Object)instance == (Object)null)
			{
				GUILayout.Label("Attach effect not initialized yet.", _small, Array.Empty<GUILayoutOption>());
			}
			else
			{
				if (string.IsNullOrWhiteSpace(_offsetRightText))
				{
					_offsetRightText = instance.TeleportEffectRightNudge.ToString("0.###");
				}
				if (string.IsNullOrWhiteSpace(_offsetFwdText))
				{
					_offsetFwdText = instance.TeleportEffectForwardNudge.ToString("0.###");
				}
				GUILayout.Label("Tweak where the attached teleport orb renders relative to the held item tip.", _small, Array.Empty<GUILayoutOption>());
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				GUILayout.Label($"Right / Left: {instance.TeleportEffectRightNudge:0.000} m", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(170f) });
				float num = GUILayout.HorizontalSlider(instance.TeleportEffectRightNudge, -2f, 2f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(220f) });
				if (Mathf.Abs(num - instance.TeleportEffectRightNudge) > 0.0005f)
				{
					instance.TeleportEffectRightNudge = num;
					_offsetRightText = instance.TeleportEffectRightNudge.ToString("0.###");
				}
				GUILayout.EndHorizontal();
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				GUILayout.Label($"Forward / Back: {instance.TeleportEffectForwardNudge:0.000} m", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(170f) });
				float num2 = GUILayout.HorizontalSlider(instance.TeleportEffectForwardNudge, -2f, 2f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(220f) });
				if (Mathf.Abs(num2 - instance.TeleportEffectForwardNudge) > 0.0005f)
				{
					instance.TeleportEffectForwardNudge = num2;
					_offsetFwdText = instance.TeleportEffectForwardNudge.ToString("0.###");
				}
				GUILayout.EndHorizontal();
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				GUILayout.Label("Manual Right/Left (m):", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(150f) });
				_offsetRightText = GUILayout.TextField(_offsetRightText, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
				if (GUILayout.Button("Apply", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
				{
					if (float.TryParse(_offsetRightText, out var result))
					{
						instance.TeleportEffectRightNudge = result;
					}
					_offsetRightText = instance.TeleportEffectRightNudge.ToString("0.###");
				}
				GUILayout.EndHorizontal();
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				GUILayout.Label("Manual Forward/Back (m):", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(150f) });
				_offsetFwdText = GUILayout.TextField(_offsetFwdText, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(80f) });
				if (GUILayout.Button("Apply", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(70f) }))
				{
					if (float.TryParse(_offsetFwdText, out var result2))
					{
						instance.TeleportEffectForwardNudge = result2;
					}
					_offsetFwdText = instance.TeleportEffectForwardNudge.ToString("0.###");
				}
				GUILayout.EndHorizontal();
				if (GUILayout.Button("Center (reset offsets)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
				{
					instance.TeleportEffectRightNudge = 0f;
					instance.TeleportEffectForwardNudge = 0f;
					_offsetRightText = "0";
					_offsetFwdText = "0";
				}
			}
		});
		DrawCard("Cave Teleport VFX (server)", delegate
		{
			//IL_0016: Unknown result type (might be due to invalid IL or missing references)
			//IL_0021: Expected O, but got Unknown
			CSRuntime cSRuntime = CSRuntime.Instance ?? CSRuntime.SpawnForHand(Actions.find_hand(right: true));
			if ((Object)cSRuntime == (Object)null)
			{
				GUILayout.Label("CSRuntime not initialized.", _small, Array.Empty<GUILayoutOption>());
			}
			else
			{
				GUILayout.Label("Spawns the cave teleport effects in front of you.", _small, Array.Empty<GUILayoutOption>());
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				if (GUILayout.Button("Surface Depart", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					cSRuntime.SpawnCaveTeleportTest(0, isDeparture: true);
				}
				if (GUILayout.Button("Surface Arrive", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					cSRuntime.SpawnCaveTeleportTest(0, isDeparture: false);
				}
				GUILayout.EndHorizontal();
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				if (GUILayout.Button("Cave Depart", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					cSRuntime.SpawnCaveTeleportTest(1, isDeparture: true);
				}
				if (GUILayout.Button("Cave Arrive", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					cSRuntime.SpawnCaveTeleportTest(1, isDeparture: false);
				}
				GUILayout.EndHorizontal();
			}
		});
		DrawSpawnAnythingCard();
		DrawProgressionCard();
		DrawWorldPacingCard();
		DrawServerSettingsCard();
		DrawVoiceCard();
	}

	private void CacheBodyBendSettings()
	{
		if (_cachedBodyBendSettings == null || _cachedBodyBendSettings.Length == 0)
		{
			_cachedBodyBendSettings = Resources.FindObjectsOfTypeAll<BodyControllerSettings>();
		}
		if (_bodyBendField == null)
		{
			_bodyBendField = AccessTools.Field(typeof(BodyControllerSettings), "bodyForwardBendMultiplier");
		}
	}

	private void DrawBodyBendMultiplierSlider()
	{
		//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d7: Expected O, but got Unknown
		CacheBodyBendSettings();
		if (_cachedBodyBendSettings == null || _cachedBodyBendSettings.Length == 0 || _bodyBendField == null)
		{
			GUILayout.Label("Body settings not available.", _small, Array.Empty<GUILayoutOption>());
			return;
		}
		float num = (float)_bodyBendField.GetValue(_cachedBodyBendSettings[0]);
		GUILayout.Label($"Twerking ({num:F1})", _h3, Array.Empty<GUILayoutOption>());
		float num2 = GUILayout.HorizontalSlider(num, -35f, 35f, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Width(390f),
			GUILayout.Height(24f)
		});
		if (!(Mathf.Abs(num2 - num) > 0.001f))
		{
			return;
		}
		BodyControllerSettings[] cachedBodyBendSettings = _cachedBodyBendSettings;
		foreach (BodyControllerSettings val in cachedBodyBendSettings)
		{
			if ((Object)val != (Object)null)
			{
				_bodyBendField.SetValue(val, num2);
			}
		}
	}

	private void DrawVoiceCard()
	{
		DrawCard("Voice (Vivox)", delegate
		{
			//IL_0074: Unknown result type (might be due to invalid IL or missing references)
			//IL_0091: Unknown result type (might be due to invalid IL or missing references)
			//IL_0096: Unknown result type (might be due to invalid IL or missing references)
			GUILayout.Label("Adjust how loud each player is locally. Uses Vivox LocalVolumeAdjustment (-10 to 50).", _small, Array.Empty<GUILayoutOption>());
			List<IPlayer> list = Player.AllPlayers?.OrderBy((IPlayer p) => p.UserInfo.Identifier).ToList() ?? new List<IPlayer>();
			if (list.Count == 0)
			{
				GUILayout.Label("No players online.", _small, Array.Empty<GUILayoutOption>());
			}
			else
			{
				_voiceScroll = GUILayout.BeginScrollView(_voiceScroll, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(200f) });
				foreach (IPlayer item in list)
				{
					GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
					GUILayout.Label($"{item.UserInfo.Username} ({item.UserInfo.Identifier})", _label, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.ExpandWidth(true) });
					int num = 0;
					try
					{
						IParticipant voice = item.Voice;
						num = ((voice != null) ? ((IParticipantProperties)voice).LocalVolumeAdjustment : 0);
					}
					catch
					{
					}
					GUILayout.Label($"{num} dB", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
					int num2 = Mathf.RoundToInt(GUILayout.HorizontalSlider((float)num, -10f, 50f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(200f) }));
					if (num2 != num)
					{
						try
						{
							if (((item != null) ? item.Voice : null) != null)
							{
								((IParticipantProperties)item.Voice).LocalVolumeAdjustment = num2;
							}
						}
						catch (Exception ex)
						{
							MelonLogger.Warning("[Unified/UI] Failed to set voice volume: " + ex.GetBaseException().Message);
						}
					}
					GUILayout.EndHorizontal();
				}
				GUILayout.EndScrollView();
			}
		});
	}

	private void DrawToggle(string label, bool state, Action<bool> onChanged)
	{
		bool flag = GUILayout.Toggle(state, label, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
		if (flag != state)
		{
			onChanged?.Invoke(flag);
		}
	}

	private void DrawCard(string title, Action body)
	{
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0048: Expected O, but got Unknown
		GUILayout.BeginVertical(_card, Array.Empty<GUILayoutOption>());
		if (!string.IsNullOrEmpty(title))
		{
			GUILayout.Label(title, _h2, Array.Empty<GUILayoutOption>());
			GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 2f), (Texture)_texDivider);
		}
		body?.Invoke();
		GUILayout.EndVertical();
		GUILayout.Space(6f);
	}

	private string GetMenuFooter()
	{
		string text = (string.IsNullOrEmpty(_statusLeft) ? "Need help? Type /help in chat." : _statusLeft);
		if (!string.IsNullOrEmpty(_statusRight))
		{
			text = text + "  |  " + _statusRight;
		}
		return text;
	}

	private void SetTeleportSphereEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportSphereEnabled, enabled);
	}

	private void SetTeleportConeEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportConeEnabled, enabled);
	}

	private void SetTeleportStarEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportStarEnabled, enabled);
	}

	private void SetTeleportRingEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportRingEnabled, enabled);
	}

	private void SetTeleportRingArenaEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportRingArenaEnabled, enabled);
		if (enabled)
		{
			ResetTeleportRingArena();
		}
	}

	private void SetTeleportCubeEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportCubeEnabled, enabled);
	}

	private void SetTeleportHelixEnabled(bool enabled)
	{
		SetTeleportShapeEnabled(ref _teleportHelixEnabled, enabled);
	}

	private void SetTeleportShapeEnabled(ref bool field, bool enabled)
	{
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_004c: Unknown result type (might be due to invalid IL or missing references)
		bool flag = AnyTeleportShapeEnabled();
		if (field != enabled)
		{
			field = enabled;
			_teleportSphereNextTime = 0f;
			bool flag2 = AnyTeleportShapeEnabled();
			if (!flag && flag2)
			{
				_teleportSphereCenterSet = false;
				_teleportSphereCenter = Vector3.zero;
			}
			else if (!flag2)
			{
				_teleportSphereCenterSet = false;
				_teleportSphereCenter = Vector3.zero;
				ResetTeleportRingArena();
			}
		}
	}

	private bool AnyTeleportShapeEnabled()
	{
		if (!_teleportSphereEnabled && !_teleportConeEnabled && !_teleportStarEnabled && !_teleportRingEnabled && !_teleportRingArenaEnabled && !_teleportCubeEnabled)
		{
			return _teleportHelixEnabled;
		}
		return true;
	}

	private void TeleportSphere_Tick()
	{
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_009c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		if (!AnyTeleportShapeEnabled() || !_sceneReady || (!_teleportSphereCenterSet && Time.time < _teleportSphereNextTime))
		{
			return;
		}
		if (!_teleportSphereCenterSet)
		{
			if (!TryGetTeleportSphereCenter(out _teleportSphereCenter))
			{
				_teleportSphereNextTime = Time.time + 1f;
				return;
			}
			_teleportSphereCenterSet = true;
		}
		if (_teleportRingArenaEnabled)
		{
			TickTeleportRingArena(_teleportSphereCenter);
		}
		if (!(Time.time < _teleportSphereNextTime))
		{
			float teleportShapeScale = GetTeleportShapeScale();
			if (_teleportSphereEnabled)
			{
				SpawnTeleportSphere(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportConeEnabled)
			{
				SpawnTeleportCone(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportStarEnabled)
			{
				SpawnTeleportStar(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportRingEnabled)
			{
				SpawnTeleportRing(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportCubeEnabled)
			{
				SpawnTeleportCube(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportHelixEnabled)
			{
				SpawnTeleportHelix(_teleportSphereCenter, teleportShapeScale);
			}
			if (_teleportSphereEnabled)
			{
				SpawnTeleportPillarsForOtherPlayers(_teleportSphereCenter);
			}
			_teleportSphereNextTime = Time.time + 1f;
		}
	}

	private float GetTeleportShapeScale()
	{
		return Mathf.Clamp(_teleportShapeScale, 0.2f, 3f);
	}

	private void ResetTeleportRingArena()
	{
		_teleportRingArenaRadius = 0f;
		_teleportRingArenaNextTime = 0f;
		_teleportRingArenaLastTime = 0f;
	}

	private void TickTeleportRingArena(Vector3 center)
	{
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		float teleportShapeScale = GetTeleportShapeScale();
		float num = 12f * teleportShapeScale;
		if (num <= 0f)
		{
			return;
		}
		float num2 = Mathf.Max(0.05f, num * 0.08f);
		if (_teleportRingArenaRadius <= 0f || _teleportRingArenaRadius > num)
		{
			_teleportRingArenaRadius = num;
		}
		float time = Time.time;
		if (!(time < _teleportRingArenaNextTime))
		{
			float num3 = ((_teleportRingArenaLastTime > 0f) ? (time - _teleportRingArenaLastTime) : 0.12f);
			_teleportRingArenaLastTime = time;
			SpawnTeleportRingArena(center, _teleportRingArenaRadius, teleportShapeScale);
			float num4 = 0.5f * teleportShapeScale;
			_teleportRingArenaRadius -= num4 * Mathf.Max(0.001f, num3);
			if (_teleportRingArenaRadius < num2)
			{
				_teleportRingArenaRadius = num2;
			}
			_teleportRingArenaNextTime = time + 0.12f;
		}
	}

	private static bool TryGetTeleportSphereCenter(out Vector3 center)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Expected O, but got Unknown
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Expected O, but got Unknown
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Expected O, but got Unknown
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		center = Vector3.zero;
		try
		{
			FriendlyOrbBehavior2 orb = Actions.get_orb();
			if ((Object)orb != (Object)null && (Object)orb.tp_rb != (Object)null)
			{
				center = orb.tp_rb.position;
				return true;
			}
		}
		catch
		{
		}
		PlayerController current = PlayerController.Current;
		if ((Object)current != (Object)null)
		{
			center = current.PlayerFeetPosition;
			return true;
		}
		return false;
	}

	private static void SpawnTeleportSphere(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportSphereOffsets = GetTeleportSphereOffsets();
			for (int i = 0; i < teleportSphereOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportSphereOffsets[i] : (teleportSphereOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportCone(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportConeOffsets = GetTeleportConeOffsets();
			for (int i = 0; i < teleportConeOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportConeOffsets[i] : (teleportConeOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportStar(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportStarOffsets = GetTeleportStarOffsets();
			for (int i = 0; i < teleportStarOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportStarOffsets[i] : (teleportStarOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportRing(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportRingOffsets = GetTeleportRingOffsets();
			for (int i = 0; i < teleportRingOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportRingOffsets[i] : (teleportRingOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportCube(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportCubeOffsets = GetTeleportCubeOffsets();
			for (int i = 0; i < teleportCubeOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportCubeOffsets[i] : (teleportCubeOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportHelix(Vector3 center, float scale)
	{
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		if (!(scale <= 0f))
		{
			List<Vector3> teleportHelixOffsets = GetTeleportHelixOffsets();
			for (int i = 0; i < teleportHelixOffsets.Count; i++)
			{
				Vector3 val = ((scale == 1f) ? teleportHelixOffsets[i] : (teleportHelixOffsets[i] * scale));
				SpawnTeleportEffectDefault(center + val);
			}
		}
	}

	private static void SpawnTeleportRingArena(Vector3 center, float radius, float scale)
	{
		//IL_0095: Unknown result type (might be due to invalid IL or missing references)
		//IL_009f: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c8: Unknown result type (might be due to invalid IL or missing references)
		float num = 0.6f * Mathf.Max(0.05f, scale);
		float num2 = 1f * scale;
		float num3 = 0.5f * scale;
		if (radius <= 0f || num <= 0f)
		{
			return;
		}
		int num4 = Mathf.Max(12, Mathf.CeilToInt((float)Math.PI * 2f * radius / num));
		bool flag = num2 > 0f && num3 > 0f;
		float num5 = num2 * 0.5f;
		for (int i = 0; i < num4; i++)
		{
			float num6 = (float)i / (float)num4 * (float)Math.PI * 2f;
			float num7 = Mathf.Cos(num6) * radius;
			float num8 = Mathf.Sin(num6) * radius;
			if (!flag)
			{
				SpawnTeleportEffectDefault(center + new Vector3(num7, 0f, num8));
				continue;
			}
			for (float num9 = 0f - num5; num9 <= num5; num9 += num3)
			{
				SpawnTeleportEffectDefault(center + new Vector3(num7, num9, num8));
			}
		}
	}

	private static void SpawnTeleportPillarsForOtherPlayers(Vector3 center)
	{
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0055: Expected O, but got Unknown
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Unknown result type (might be due to invalid IL or missing references)
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		List<IPlayer> list = Player.AllPlayers?.ToList();
		if (list == null || list.Count == 0)
		{
			return;
		}
		float num = 400f;
		foreach (IPlayer item in list)
		{
			if (item == null || item.IsLocalPlayer)
			{
				continue;
			}
			PlayerController playerController = item.PlayerController;
			if (!((Object)playerController == (Object)null))
			{
				Vector3 playerFeetPosition = playerController.PlayerFeetPosition;
				Vector3 val = playerFeetPosition - center;
				if (!(((Vector3)(ref val)).sqrMagnitude > num))
				{
					SpawnTeleportPillar(playerFeetPosition);
				}
			}
		}
	}

	private static void SpawnTeleportPillar(Vector3 basePos)
	{
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		float num = 4f;
		float num2 = 0.5f;
		if (!(num <= 0f) && !(num2 <= 0f))
		{
			for (float num3 = 0f; num3 <= num; num3 += num2)
			{
				SpawnTeleportEffectDefault(basePos + Vector3.up * num3);
			}
		}
	}

	private static List<Vector3> GetTeleportConeOffsets()
	{
		//IL_008b: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e3: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportConeOffsets != null)
		{
			return _teleportConeOffsets;
		}
		float num = 18f;
		float num2 = 10f;
		float num3 = 1f;
		List<Vector3> list = new List<Vector3>();
		if (num <= 0f || num2 <= 0f || num3 <= 0f)
		{
			_teleportConeOffsets = list;
			return list;
		}
		for (float num4 = 0f; num4 <= num; num4 += num3)
		{
			float num5 = ((num <= 0f) ? 0f : (num4 / num));
			float num6 = Mathf.Lerp(0f, num2, num5);
			if (num6 <= 0.001f)
			{
				list.Add(new Vector3(0f, num4, 0f));
				continue;
			}
			float num7 = (float)Math.PI * 2f * num6;
			int num8 = Mathf.Max(6, Mathf.CeilToInt(num7 / num3));
			for (int i = 0; i < num8; i++)
			{
				float num9 = (float)i / (float)num8 * (float)Math.PI * 2f;
				list.Add(new Vector3(Mathf.Cos(num9) * num6, num4, Mathf.Sin(num9) * num6));
			}
		}
		_teleportConeOffsets = list;
		return list;
	}

	private static List<Vector3> GetTeleportStarOffsets()
	{
		//IL_009f: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00db: Unknown result type (might be due to invalid IL or missing references)
		//IL_00dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_0112: Unknown result type (might be due to invalid IL or missing references)
		//IL_0114: Unknown result type (might be due to invalid IL or missing references)
		//IL_0118: Unknown result type (might be due to invalid IL or missing references)
		//IL_011d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0121: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportStarOffsets != null)
		{
			return _teleportStarOffsets;
		}
		float num = 12f;
		float num2 = 6f;
		float num3 = 1f;
		float thickness = 1f;
		float layerSpacing = 0.5f;
		List<Vector3> list = new List<Vector3>();
		if (num <= 0f || num2 <= 0f || num3 <= 0f)
		{
			_teleportStarOffsets = list;
			return list;
		}
		List<Vector3> list2 = new List<Vector3>();
		for (int i = 0; i < 10; i++)
		{
			float num4 = (float)i / 10f * (float)Math.PI * 2f;
			float num5 = ((i % 2 == 0) ? num : num2);
			list2.Add(new Vector3(Mathf.Cos(num4) * num5, 0f, Mathf.Sin(num4) * num5));
		}
		for (int j = 0; j < list2.Count; j++)
		{
			Vector3 val = list2[j];
			Vector3 val2 = list2[(j + 1) % list2.Count];
			float num6 = Vector3.Distance(val, val2);
			int num7 = Mathf.Max(1, Mathf.CeilToInt(num6 / num3));
			for (int k = 0; k <= num7; k++)
			{
				float num8 = ((num7 == 0) ? 0f : ((float)k / (float)num7));
				Vector3 basePos = Vector3.Lerp(val, val2, num8);
				AddLayeredOffset(list, basePos, thickness, layerSpacing);
			}
		}
		_teleportStarOffsets = list;
		return list;
	}

	private static List<Vector3> GetTeleportRingOffsets()
	{
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_009f: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportRingOffsets != null)
		{
			return _teleportRingOffsets;
		}
		float num = 12f;
		float num2 = 0.6f;
		float thickness = 1f;
		float layerSpacing = 0.5f;
		List<Vector3> list = new List<Vector3>();
		if (num <= 0f || num2 <= 0f)
		{
			_teleportRingOffsets = list;
			return list;
		}
		int num3 = Mathf.Max(12, Mathf.CeilToInt((float)Math.PI * 2f * num / num2));
		Vector3 basePos = default(Vector3);
		for (int i = 0; i < num3; i++)
		{
			float num4 = (float)i / (float)num3 * (float)Math.PI * 2f;
			((Vector3)(ref basePos))._002Ector(Mathf.Cos(num4) * num, 0f, Mathf.Sin(num4) * num);
			AddLayeredOffset(list, basePos, thickness, layerSpacing);
		}
		_teleportRingOffsets = list;
		return list;
	}

	private static List<Vector3> GetTeleportCubeOffsets()
	{
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0077: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_0118: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0123: Unknown result type (might be due to invalid IL or missing references)
		//IL_0125: Unknown result type (might be due to invalid IL or missing references)
		//IL_012e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0130: Unknown result type (might be due to invalid IL or missing references)
		//IL_0139: Unknown result type (might be due to invalid IL or missing references)
		//IL_013b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0144: Unknown result type (might be due to invalid IL or missing references)
		//IL_0146: Unknown result type (might be due to invalid IL or missing references)
		//IL_014f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0151: Unknown result type (might be due to invalid IL or missing references)
		//IL_015a: Unknown result type (might be due to invalid IL or missing references)
		//IL_015c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0165: Unknown result type (might be due to invalid IL or missing references)
		//IL_0167: Unknown result type (might be due to invalid IL or missing references)
		//IL_0170: Unknown result type (might be due to invalid IL or missing references)
		//IL_0172: Unknown result type (might be due to invalid IL or missing references)
		//IL_017b: Unknown result type (might be due to invalid IL or missing references)
		//IL_017d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0186: Unknown result type (might be due to invalid IL or missing references)
		//IL_0188: Unknown result type (might be due to invalid IL or missing references)
		//IL_0191: Unknown result type (might be due to invalid IL or missing references)
		//IL_0193: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportCubeOffsets != null)
		{
			return _teleportCubeOffsets;
		}
		float num = 16f;
		float num2 = 1f;
		List<Vector3> list = new List<Vector3>();
		if (num <= 0f || num2 <= 0f)
		{
			_teleportCubeOffsets = list;
			return list;
		}
		float num3 = num * 0.5f;
		Vector3 val = default(Vector3);
		((Vector3)(ref val))._002Ector(0f - num3, num3, 0f - num3);
		Vector3 val2 = default(Vector3);
		((Vector3)(ref val2))._002Ector(num3, num3, 0f - num3);
		Vector3 val3 = default(Vector3);
		((Vector3)(ref val3))._002Ector(num3, num3, num3);
		Vector3 val4 = default(Vector3);
		((Vector3)(ref val4))._002Ector(0f - num3, num3, num3);
		Vector3 val5 = default(Vector3);
		((Vector3)(ref val5))._002Ector(0f - num3, 0f - num3, 0f - num3);
		Vector3 val6 = default(Vector3);
		((Vector3)(ref val6))._002Ector(num3, 0f - num3, 0f - num3);
		Vector3 val7 = default(Vector3);
		((Vector3)(ref val7))._002Ector(num3, 0f - num3, num3);
		Vector3 val8 = default(Vector3);
		((Vector3)(ref val8))._002Ector(0f - num3, 0f - num3, num3);
		AddEdgeOffsets(list, val, val2, num2);
		AddEdgeOffsets(list, val2, val3, num2);
		AddEdgeOffsets(list, val3, val4, num2);
		AddEdgeOffsets(list, val4, val, num2);
		AddEdgeOffsets(list, val5, val6, num2);
		AddEdgeOffsets(list, val6, val7, num2);
		AddEdgeOffsets(list, val7, val8, num2);
		AddEdgeOffsets(list, val8, val5, num2);
		AddEdgeOffsets(list, val5, val, num2);
		AddEdgeOffsets(list, val6, val2, num2);
		AddEdgeOffsets(list, val7, val3, num2);
		AddEdgeOffsets(list, val8, val4, num2);
		_teleportCubeOffsets = list;
		return list;
	}

	private static List<Vector3> GetTeleportHelixOffsets()
	{
		//IL_00a0: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportHelixOffsets != null)
		{
			return _teleportHelixOffsets;
		}
		float num = 16f;
		float num2 = 6f;
		float num3 = 3f;
		float num4 = 0.5f;
		List<Vector3> list = new List<Vector3>();
		if (num <= 0f || num2 <= 0f || num4 <= 0f)
		{
			_teleportHelixOffsets = list;
			return list;
		}
		int num5 = Mathf.Max(1, Mathf.CeilToInt(num / num4));
		for (int i = 0; i <= num5; i++)
		{
			float num6 = ((num5 == 0) ? 0f : ((float)i / (float)num5));
			float num7 = num6 * num;
			float num8 = num6 * (float)Math.PI * 2f * num3;
			list.Add(new Vector3(Mathf.Cos(num8) * num2, num7, Mathf.Sin(num8) * num2));
		}
		_teleportHelixOffsets = list;
		return list;
	}

	private static void AddLayeredOffset(List<Vector3> offsets, Vector3 basePos, float thickness, float layerSpacing)
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		if (thickness <= 0f || layerSpacing <= 0f)
		{
			offsets.Add(basePos);
			return;
		}
		float num = thickness * 0.5f;
		for (float num2 = 0f - num; num2 <= num; num2 += layerSpacing)
		{
			offsets.Add(new Vector3(basePos.x, basePos.y + num2, basePos.z));
		}
	}

	private static void AddEdgeOffsets(List<Vector3> offsets, Vector3 start, Vector3 end, float spacing)
	{
		//IL_0000: Unknown result type (might be due to invalid IL or missing references)
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_002f: Unknown result type (might be due to invalid IL or missing references)
		float num = Vector3.Distance(start, end);
		int num2 = Mathf.Max(1, Mathf.CeilToInt(num / spacing));
		for (int i = 0; i <= num2; i++)
		{
			float num3 = ((num2 == 0) ? 0f : ((float)i / (float)num2));
			offsets.Add(Vector3.Lerp(start, end, num3));
		}
	}

	private static List<Vector3> GetTeleportSphereOffsets()
	{
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_007d: Unknown result type (might be due to invalid IL or missing references)
		if (_teleportSphereOffsets != null)
		{
			return _teleportSphereOffsets;
		}
		float num = 20f;
		float num2 = 1f;
		float num3 = 0.5f;
		float num4 = num - num3;
		float num5 = num4 * num4;
		float num6 = num * num;
		List<Vector3> list = new List<Vector3>();
		Vector3 item = default(Vector3);
		for (float num7 = 0f - num; num7 <= num; num7 += num2)
		{
			for (float num8 = 0f - num; num8 <= num; num8 += num2)
			{
				for (float num9 = 0f - num; num9 <= num; num9 += num2)
				{
					((Vector3)(ref item))._002Ector(num7, num8, num9);
					float sqrMagnitude = ((Vector3)(ref item)).sqrMagnitude;
					if (sqrMagnitude >= num5 && sqrMagnitude <= num6)
					{
						list.Add(item);
					}
				}
			}
		}
		_teleportSphereOffsets = list;
		return list;
	}

	private static void SpawnTeleportEffectDefault(Vector3 position)
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Expected O, but got Unknown
		//IL_0098: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		try
		{
			PlayerController current = PlayerController.Current;
			if ((Object)current == (Object)null)
			{
				return;
			}
			PlayerEffectController component = ((Component)((Component)current).transform).GetComponent<PlayerEffectController>();
			if ((Object)component == (Object)null)
			{
				return;
			}
			object obj = AccessTools.Field(((object)component).GetType(), "remoteTeleportEffect")?.GetValue(component);
			if (obj == null)
			{
				return;
			}
			Type type = AccessTools.Inner(((object)component).GetType(), "PlayerEffectTeleportInformation");
			if (type == null)
			{
				return;
			}
			object obj2 = Activator.CreateInstance(type);
			AccessTools.Field(type, "teleportEffectPosition")?.SetValue(obj2, position);
			AccessTools.Field(type, "teleportType")?.SetValue(obj2, TeleportType.Unknown);
			MethodInfo method = obj.GetType().GetMethod("SendToChunks");
			if (!(method == null))
			{
				method.Invoke(obj, new object[2]
				{
					obj2,
					Array.Empty<IPlayer>()
				});
				if (CSRuntime.SmiteEnabled)
				{
					CSRuntime.TrySmiteAt(position);
				}
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Teleport sphere effect failed: " + ex.GetBaseException().Message);
		}
	}

	public void SetInvisEnabled(bool enable)
	{
		if (_invisEnabled != enable)
		{
			_invisEnabled = enable;
			if (_invisEnabled)
			{
				_invisNextScanAt = 0f;
				_invisRoot = null;
				_invisPlayer = null;
				_invisOriginalStates.Clear();
				ApplyInvisibility(forceScan: true);
			}
			else
			{
				RestoreInvisibility();
			}
		}
	}

	private void Invisibility_Tick()
	{
		if (_invisEnabled && _sceneReady)
		{
			ApplyInvisibility(forceScan: false);
		}
	}

	private void ApplyInvisibility(bool forceScan)
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Expected O, but got Unknown
		//IL_002b: Expected O, but got Unknown
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_004b: Expected O, but got Unknown
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Expected O, but got Unknown
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c9: Expected O, but got Unknown
		PlayerController current = PlayerController.Current;
		if ((Object)current == (Object)null)
		{
			return;
		}
		if ((Object)current != (Object)_invisPlayer)
		{
			RestoreInvisibility();
			_invisPlayer = current;
		}
		if ((Object)_invisRoot == (Object)null)
		{
			_invisRoot = FindInvisRoot(((Component)current).transform);
		}
		if ((Object)_invisRoot == (Object)null)
		{
			return;
		}
		if (forceScan || Time.time >= _invisNextScanAt)
		{
			CacheInvisRenderers(_invisRoot);
			_invisNextScanAt = Time.time + 0.5f;
		}
		foreach (KeyValuePair<Renderer, bool> invisOriginalState in _invisOriginalStates)
		{
			Renderer key = invisOriginalState.Key;
			if ((Object)key != (Object)null)
			{
				key.enabled = false;
			}
		}
	}

	private void CacheInvisRenderers(Transform root)
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Expected O, but got Unknown
		Renderer[] componentsInChildren = ((Component)root).GetComponentsInChildren<Renderer>(true);
		foreach (Renderer val in componentsInChildren)
		{
			if (!((Object)val == (Object)null) && !_invisOriginalStates.ContainsKey(val))
			{
				_invisOriginalStates[val] = val.enabled;
			}
		}
	}

	private void RestoreInvisibility()
	{
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Expected O, but got Unknown
		foreach (KeyValuePair<Renderer, bool> invisOriginalState in _invisOriginalStates)
		{
			Renderer key = invisOriginalState.Key;
			if ((Object)key != (Object)null)
			{
				key.enabled = invisOriginalState.Value;
			}
		}
		_invisOriginalStates.Clear();
		_invisRoot = null;
		_invisPlayer = null;
		_invisNextScanAt = 0f;
	}

	private static Transform FindInvisRoot(Transform root)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		if ((Object)root == (Object)null)
		{
			return null;
		}
		Transform val = FindChildByNameContains(root, "VR Player Character");
		if ((Object)val != (Object)null)
		{
			return val;
		}
		val = FindChildByNameContains(root, "Player Character");
		return val ?? root;
	}

	private static Transform FindChildByNameContains(Transform root, string frag)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)root == (Object)null || string.IsNullOrEmpty(frag))
		{
			return null;
		}
		Queue<Transform> queue = new Queue<Transform>();
		queue.Enqueue(root);
		while (queue.Count > 0)
		{
			Transform val = queue.Dequeue();
			string name = ((Object)val).name;
			if (!string.IsNullOrEmpty(name) && name.IndexOf(frag, StringComparison.OrdinalIgnoreCase) >= 0)
			{
				return val;
			}
			for (int i = 0; i < val.childCount; i++)
			{
				queue.Enqueue(val.GetChild(i));
			}
		}
		return null;
	}

	public void ToggleThirdPersonCamera()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0101: Unknown result type (might be due to invalid IL or missing references)
		//IL_010c: Expected O, but got Unknown
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Expected O, but got Unknown
		//IL_0122: Unknown result type (might be due to invalid IL or missing references)
		//IL_012d: Expected O, but got Unknown
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_007f: Expected O, but got Unknown
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_006c: Expected O, but got Unknown
		PlayerController current = PlayerController.Current;
		if ((Object)current == (Object)null)
		{
			return;
		}
		thirdpersonMode = !thirdpersonMode;
		if (thirdpersonMode)
		{
			if ((Object)thirdPersonCam == (Object)null)
			{
				thirdPersonCam = new GameObject("ThirdPersonCam").AddComponent<Camera>();
				Object.DontDestroyOnLoad((Object)((Component)thirdPersonCam).gameObject);
			}
			Camera camera = current.Camera;
			if ((Object)camera != (Object)null)
			{
				thirdPersonCam.fieldOfView = camera.fieldOfView;
				thirdPersonCam.nearClipPlane = camera.nearClipPlane;
				thirdPersonCam.farClipPlane = camera.farClipPlane;
				thirdPersonCam.cullingMask = camera.cullingMask;
				thirdPersonCam.allowHDR = camera.allowHDR;
				thirdPersonCam.allowMSAA = camera.allowMSAA;
				((Behaviour)camera).enabled = false;
			}
			((Behaviour)thirdPersonCam).enabled = true;
		}
		else
		{
			if ((Object)thirdPersonCam != (Object)null)
			{
				((Behaviour)thirdPersonCam).enabled = false;
			}
			Camera camera2 = current.Camera;
			if ((Object)camera2 != (Object)null)
			{
				((Behaviour)camera2).enabled = true;
			}
		}
	}

	private void HandleThirdPersonCamera()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_0043: Expected O, but got Unknown
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_007f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0084: Unknown result type (might be due to invalid IL or missing references)
		//IL_008e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0093: Unknown result type (might be due to invalid IL or missing references)
		PlayerController current = PlayerController.Current;
		if (!((Object)current == (Object)null) && !((Object)thirdPersonCam == (Object)null) && thirdpersonMode && (Object)current.Camera != (Object)null)
		{
			Transform transform = ((Component)current).transform;
			((Component)thirdPersonCam).transform.position = transform.position + transform.TransformDirection(thirdpersonOffset);
			((Component)thirdPersonCam).transform.LookAt(transform.position + Vector3.up * 0.5f);
		}
	}

	private void ThirdPerson_Tick()
	{
		HandleThirdPersonCamera();
	}

	private string GetFollowModeLabel()
	{
		if (snapFollowActive)
		{
			return "Following";
		}
		if (snapFollowPrimed)
		{
			return "Point and Grab to Follow";
		}
		return "Follow Player";
	}

	public void ToggleFollowMode()
	{
		if (!snapFollowActive && !snapFollowPrimed)
		{
			snapFollowPrimed = true;
			snapFollowActive = false;
			return;
		}
		snapFollowPrimed = false;
		snapFollowActive = false;
		snapFollowTarget = null;
		snapFollowHeadAnchor = null;
	}

	public void SelectFollowTarget(bool isRightHand)
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Expected O, but got Unknown
		//IL_0043: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_0080: Expected O, but got Unknown
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0096: Expected O, but got Unknown
		//IL_0096: Expected O, but got Unknown
		//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d2: Expected O, but got Unknown
		//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f4: Expected O, but got Unknown
		PlayerController current = PlayerController.Current;
		if ((Object)current == (Object)null)
		{
			return;
		}
		Controller val = (isRightHand ? current.RightController : current.LeftController);
		RaycastHit val2 = default(RaycastHit);
		if ((Object)val == (Object)null || !Physics.Raycast(new Ray(((Component)val).transform.position, ((Component)val).transform.forward), ref val2, 100f))
		{
			return;
		}
		PlayerController componentInParent = ((Component)((RaycastHit)(ref val2)).collider).GetComponentInParent<PlayerController>();
		if ((Object)componentInParent != (Object)null && (Object)componentInParent != (Object)current)
		{
			snapFollowTarget = ((Component)componentInParent).transform;
			snapFollowPrimed = false;
			snapFollowActive = true;
			Transform val3 = FindChildByName(snapFollowTarget, "Remote Head");
			if ((Object)val3 == (Object)null)
			{
				val3 = FindChildByName(snapFollowTarget, "Eating Mouth");
			}
			if ((Object)val3 == (Object)null)
			{
				val3 = snapFollowTarget;
			}
			snapFollowHeadAnchor = val3;
		}
	}

	private void FollowMode_Tick()
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Expected O, but got Unknown
		//IL_00ee: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f9: Expected O, but got Unknown
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Expected O, but got Unknown
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Expected O, but got Unknown
		//IL_0054: Unknown result type (might be due to invalid IL or missing references)
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_009d: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d5: Unknown result type (might be due to invalid IL or missing references)
		if (snapFollowActive && (Object)snapFollowTarget != (Object)null)
		{
			PlayerController current = PlayerController.Current;
			if ((Object)current != (Object)null && (Object)snapFollowHeadAnchor != (Object)null)
			{
				Vector3 val = snapFollowHeadAnchor.position + snapFollowHeadAnchor.TransformDirection(snapFollowOffset);
				((Component)current).transform.position = Vector3.Lerp(((Component)current).transform.position, val, Time.deltaTime * 5f);
				Quaternion val2 = Quaternion.LookRotation(snapFollowHeadAnchor.position - ((Component)current).transform.position);
				((Component)current).transform.rotation = Quaternion.Slerp(((Component)current).transform.rotation, val2, Time.deltaTime * 5f);
			}
		}
		else if (snapFollowActive && (Object)snapFollowTarget == (Object)null)
		{
			snapFollowActive = false;
		}
	}

	private Transform FindChildByName(Transform parent, string name)
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0015: Expected O, but got Unknown
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_003c: Expected O, but got Unknown
		foreach (Transform item in parent)
		{
			Transform val = item;
			if (((Object)val).name == name)
			{
				return val;
			}
			Transform val2 = FindChildByName(val, name);
			if ((Object)val2 != (Object)null)
			{
				return val2;
			}
		}
		return null;
	}

	private void PartnerThrow_Tick()
	{
		if (!_hold.Active)
		{
			ApplyPendingThrow();
		}
		HandlePartnerHold();
	}

	private string GetPartnerHoldLabel()
	{
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Expected O, but got Unknown
		if (_hold.Active)
		{
			return "Being Held :3";
		}
		if ((Object)_partnerRoot != (Object)null)
		{
			return "Partner Selected";
		}
		if (_partnerHoldPrimed)
		{
			return "Pick Player...";
		}
		return "Partner Hold";
	}

	private void TogglePartnerHoldPrimed()
	{
		if (!_hold.Active)
		{
			_partnerHoldPrimed = !_partnerHoldPrimed;
		}
	}

	private void HandlePartnerHold()
	{
		//IL_013c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0147: Expected O, but got Unknown
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Expected O, but got Unknown
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_005b: Expected O, but got Unknown
		//IL_006b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0070: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_0092: Unknown result type (might be due to invalid IL or missing references)
		//IL_009d: Expected O, but got Unknown
		//IL_00b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c0: Expected O, but got Unknown
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00dd: Expected O, but got Unknown
		//IL_00dd: Expected O, but got Unknown
		if (_partnerHoldPrimed && Mouse.current != null && Mouse.current.leftButton.wasPressedThisFrame)
		{
			PlayerController current = PlayerController.Current;
			Camera val = (((Object)current != (Object)null) ? current.Camera : null);
			RaycastHit val2 = default(RaycastHit);
			if ((Object)val != (Object)null && Physics.Raycast(val.ScreenPointToRay(Vector2.op_Implicit(((InputControl<Vector2>)(object)((Pointer)Mouse.current).position).ReadValue())), ref val2, 100f))
			{
				Transform val3 = (((Object)((RaycastHit)(ref val2)).collider != (Object)null) ? ((Component)((RaycastHit)(ref val2)).collider).transform.root : null);
				if ((Object)val3 != (Object)null && (Object)val3 != (Object)((Component)current).transform.root && ((Object)val3).name.Contains("VR Player"))
				{
					_partnerRoot = val3;
					_partnerLeftHand = FindChildByNameFast(val3, "Left Hand");
					_partnerRightHand = FindChildByNameFast(val3, "Right Hand");
					_partnerHoldPrimed = false;
					MelonLogger.Msg("[PartnerHold] Selected " + ((Object)val3).name);
				}
			}
		}
		if (!((Object)_partnerRoot == (Object)null))
		{
			if (!_hold.Active)
			{
				TryBeginHold();
			}
			if (_hold.Active)
			{
				MaintainHold();
				TryReleaseHold();
			}
		}
	}

	private Transform[] GetMySlots()
	{
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Expected O, but got Unknown
		//IL_002e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Expected O, but got Unknown
		if (_mySlots != null)
		{
			return _mySlots;
		}
		PlayerController current = PlayerController.Current;
		Transform val = (((Object)current != (Object)null) ? ((Component)current).transform : null);
		if ((Object)val == (Object)null)
		{
			return _mySlots = Array.Empty<Transform>();
		}
		List<Transform> list = new List<Transform>(8);
		Transform[] componentsInChildren = ((Component)val).GetComponentsInChildren<Transform>(true);
		foreach (Transform val2 in componentsInChildren)
		{
			string name = ((Object)val2).name;
			for (int j = 0; j < PartnerSlotNeedles.Length; j++)
			{
				if (name.IndexOf(PartnerSlotNeedles[j], StringComparison.OrdinalIgnoreCase) >= 0)
				{
					list.Add(val2);
					break;
				}
			}
		}
		_mySlots = list.ToArray();
		return _mySlots;
	}

	private bool TryNearestSlot(Transform hand, out Transform slot, float radius = 0.5f)
	{
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Expected O, but got Unknown
		//IL_0037: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		slot = null;
		Transform[] mySlots = GetMySlots();
		if ((Object)(object)hand == (Object)null || mySlots == null || mySlots.Length == 0)
		{
			return false;
		}
		float num = radius * radius;
		Transform[] array = mySlots;
		foreach (Transform val in array)
		{
			if ((Object)(object)val != (Object)null)
			{
				Vector3 val2 = val.position - hand.position;
				float sqrMagnitude = ((Vector3)(ref val2)).sqrMagnitude;
				if (sqrMagnitude <= num)
				{
					num = sqrMagnitude;
					slot = val;
				}
			}
		}
		return (Object)slot != (Object)null;
	}

	private void TryBeginHold()
	{
		if ((Object)(object)_partnerLeftHand == (Object)null && (Object)(object)_partnerRightHand == (Object)null)
		{
			_partnerLeftHand = FindChildByNameFast(_partnerRoot, "Left Hand");
			_partnerRightHand = FindChildByNameFast(_partnerRoot, "Right Hand");
			if ((Object)(object)_partnerLeftHand == (Object)null && (Object)(object)_partnerRightHand == (Object)null)
			{
				return;
			}
		}
		Transform slot2;
		if ((Object)(object)_partnerLeftHand != (Object)null && TryNearestSlot(_partnerLeftHand, out var slot) && (GetRemoteGripState(_partnerRoot) & GripFlags.Left) != GripFlags.None)
		{
			BeginHold(isRight: false, _partnerLeftHand, slot);
		}
		else if ((Object)(object)_partnerRightHand != (Object)null && TryNearestSlot(_partnerRightHand, out slot2) && (GetRemoteGripState(_partnerRoot) & GripFlags.Right) != GripFlags.None)
		{
			BeginHold(isRight: true, _partnerRightHand, slot2);
		}
	}

	private bool TryGetPartnerLook(out Vector3 fwd)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Expected O, but got Unknown
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Expected O, but got Unknown
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ac: Unknown result type (might be due to invalid IL or missing references)
		fwd = Vector3.zero;
		if ((Object)_partnerRoot == (Object)null)
		{
			return false;
		}
		Transform val = FindChildByNameFast(_partnerRoot, "Remote Head");
		if ((Object)val == (Object)null)
		{
			val = FindChildByNameFast(_partnerRoot, "Head") ?? FindChildByNameFast(_partnerRoot, "Eating Mouth");
		}
		if ((Object)(object)val != (Object)null)
		{
			fwd = val.forward;
			return true;
		}
		if (_orbit.Active && (Object)(object)_orbit.ControlHand != (Object)null)
		{
			fwd = _orbit.ControlHand.forward;
			return true;
		}
		fwd = _partnerRoot.forward;
		return true;
	}

	private void BeginHold(bool isRight, Transform hand, Transform mySlot)
	{
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Expected O, but got Unknown
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_0093: Unknown result type (might be due to invalid IL or missing references)
		//IL_0098: Unknown result type (might be due to invalid IL or missing references)
		//IL_009e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a9: Expected O, but got Unknown
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Expected O, but got Unknown
		//IL_00e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_012d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0132: Unknown result type (might be due to invalid IL or missing references)
		//IL_015c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0162: Unknown result type (might be due to invalid IL or missing references)
		PlayerController current = PlayerController.Current;
		PlayerController val = ((current is SmoothLocomotionPlayerController) ? current : null);
		SmoothLocomotion val2 = (((Object)(object)val != (Object)null) ? ((SmoothLocomotionPlayerController)val).SmoothLocomotion : null);
		_hold.Active = true;
		_hold.IsRight = isRight;
		_hold.Hand = hand;
		_hold.MySlot = mySlot;
		_hold.HasLastHandPos = true;
		_hold.LastHandPos = (((Object)hand != (Object)null) ? hand.position : Vector3.zero);
		_hold.HandVelocity = Vector3.zero;
		float num = (((Object)val2 != (Object)null) ? val2.BodyCenter.y : (((Object)current != (Object)null) ? current.BodyCenter.y : 0f));
		_hold.BodyYOffset = num - hand.position.y;
		_orbit.Active = true;
		_orbit.CenterHand = hand;
		_orbit.ControlHand = (isRight ? _partnerLeftHand : _partnerRightHand);
		_throwActive = false;
		_throwVelocity = Vector3.zero;
		_throwTimeLeft = 0f;
		float radius = 1.2f;
		if ((Object)(object)mySlot != (Object)null && (Object)(object)hand != (Object)null)
		{
			radius = Mathf.Clamp(Vector3.Distance(mySlot.position, hand.position), 0.3f, 4f);
		}
		_orbit.Radius = radius;
		MelonLogger.Msg(string.Format("[PartnerHold] Grabbed by {0} at '{1}' (orbit r={2:0.00})", isRight ? "Right" : "Left", ((Object)mySlot).name, _orbit.Radius));
		ApplyOrbitPosition(immediate: true);
	}

	private void MaintainHold()
	{
		if (!_hold.Active || !_orbit.Active || (Object)(object)_hold.MySlot == (Object)null || (Object)(object)_orbit.CenterHand == (Object)null)
		{
			CancelHold();
			return;
		}
		UpdateHandVelocity();
		_orbit.ControlHand = (_hold.IsRight ? _partnerLeftHand : _partnerRightHand);
		GripFlags remoteGripState = GetRemoteGripState(_partnerRoot);
		if (_hold.IsRight ? ((remoteGripState & GripFlags.Left) > GripFlags.None) : ((remoteGripState & GripFlags.Right) > GripFlags.None))
		{
			_orbit.Radius = Mathf.Clamp(_orbit.Radius + 1.8f * Time.deltaTime, 0.3f, 4f);
			ApplyOrbitPosition(immediate: false, forceLookBias: true);
		}
		else
		{
			ApplyOrbitPosition(immediate: false);
		}
	}

	private void ApplyOrbitPosition(bool immediate, bool forceLookBias = false)
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_006c: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00db: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_0115: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0130: Unknown result type (might be due to invalid IL or missing references)
		//IL_0135: Unknown result type (might be due to invalid IL or missing references)
		//IL_0145: Unknown result type (might be due to invalid IL or missing references)
		//IL_014a: Unknown result type (might be due to invalid IL or missing references)
		//IL_014f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0152: Unknown result type (might be due to invalid IL or missing references)
		//IL_015d: Expected O, but got Unknown
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_016d: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ee: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		//IL_0104: Unknown result type (might be due to invalid IL or missing references)
		//IL_0109: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_020b: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_0204: Unknown result type (might be due to invalid IL or missing references)
		//IL_020d: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bb: Expected O, but got Unknown
		//IL_022c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_01be: Unknown result type (might be due to invalid IL or missing references)
		//IL_023a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0245: Expected O, but got Unknown
		//IL_0255: Unknown result type (might be due to invalid IL or missing references)
		//IL_0260: Expected O, but got Unknown
		//IL_026f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0274: Unknown result type (might be due to invalid IL or missing references)
		//IL_0276: Unknown result type (might be due to invalid IL or missing references)
		//IL_0264: Unknown result type (might be due to invalid IL or missing references)
		//IL_0266: Unknown result type (might be due to invalid IL or missing references)
		PlayerController current = PlayerController.Current;
		Transform val = (((Object)current != (Object)null) ? ((Component)current).transform : null);
		if ((Object)(object)val == (Object)null || (Object)(object)_hold.MySlot == (Object)null || (Object)(object)_orbit.CenterHand == (Object)null)
		{
			return;
		}
		Vector3 val2;
		if ((Object)(object)_orbit.ControlHand != (Object)null)
		{
			val2 = _orbit.ControlHand.position - _orbit.CenterHand.position;
			if (((Vector3)(ref val2)).sqrMagnitude < 1E-06f)
			{
				val2 = (((Object)(object)_partnerRoot != (Object)null) ? _partnerRoot.forward : Vector3.forward);
			}
		}
		else
		{
			val2 = (((Object)(object)_partnerRoot != (Object)null) ? _partnerRoot.forward : Vector3.forward);
		}
		val2 = ((Vector3)(ref val2)).normalized;
		if (forceLookBias && TryGetPartnerLook(out var fwd))
		{
			Vector3 val3 = Vector3.Slerp(val2, ((Vector3)(ref fwd)).normalized, 0.35f);
			val2 = ((Vector3)(ref val3)).normalized;
		}
		Vector3 val4 = _orbit.CenterHand.position + val2 * Mathf.Max(_orbit.Radius, 0.3f) - _hold.MySlot.position;
		if ((Object)current != (Object)null)
		{
			float num = _orbit.CenterHand.position.y + _hold.BodyYOffset;
			PlayerController val5 = ((current is SmoothLocomotionPlayerController) ? current : null);
			SmoothLocomotion val6 = (((Object)(object)val5 != (Object)null) ? ((SmoothLocomotionPlayerController)val5).SmoothLocomotion : null);
			float num2 = (((Object)val6 != (Object)null) ? val6.BodyCenter.y : current.BodyCenter.y);
			val4.y = num - num2;
		}
		Vector3 val7 = (immediate ? val4 : Vector3.Lerp(Vector3.zero, val4, 1f - Mathf.Exp(-18f * Time.deltaTime)));
		PlayerController val8 = ((current is SmoothLocomotionPlayerController) ? current : null);
		SmoothLocomotion val9 = (((Object)(object)val8 != (Object)null) ? ((SmoothLocomotionPlayerController)val8).SmoothLocomotion : null);
		CharacterController val10 = (((Object)val9 != (Object)null) ? val9.CharacterController : null);
		if ((Object)val10 != (Object)null)
		{
			val10.Move(val7);
		}
		else
		{
			val.position += val7;
		}
	}

	private void TryReleaseHold()
	{
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Expected O, but got Unknown
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		GripFlags remoteGripState = GetRemoteGripState(_partnerRoot);
		if (!(_hold.IsRight ? ((remoteGripState & GripFlags.Right) > GripFlags.None) : ((remoteGripState & GripFlags.Left) > GripFlags.None)))
		{
			MelonLogger.Msg("[PartnerHold] Released.");
			PlayerController current = PlayerController.Current;
			if ((Object)current != (Object)null)
			{
				StartThrow(current, _hold.HandVelocity);
			}
			CancelHold();
		}
	}

	private void CancelHold()
	{
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Unknown result type (might be due to invalid IL or missing references)
		_hold.Active = false;
		_hold.Hand = null;
		_hold.MySlot = null;
		_hold.HasLastHandPos = false;
		_hold.HandVelocity = Vector3.zero;
		_hold.BodyYOffset = 0f;
	}

	private void UpdateHandVelocity()
	{
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Expected O, but got Unknown
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0085: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0090: Unknown result type (might be due to invalid IL or missing references)
		//IL_0095: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Unknown result type (might be due to invalid IL or missing references)
		//IL_0073: Unknown result type (might be due to invalid IL or missing references)
		//IL_0078: Unknown result type (might be due to invalid IL or missing references)
		if (!_hold.Active || (Object)_hold.Hand == (Object)null)
		{
			return;
		}
		float deltaTime = Time.deltaTime;
		if (!(deltaTime <= 0f))
		{
			Vector3 position = _hold.Hand.position;
			if (!_hold.HasLastHandPos)
			{
				_hold.HasLastHandPos = true;
				_hold.LastHandPos = position;
				_hold.HandVelocity = Vector3.zero;
			}
			else
			{
				Vector3 val = (position - _hold.LastHandPos) / deltaTime;
				float num = 1f - Mathf.Exp(-14f * deltaTime);
				_hold.HandVelocity = Vector3.Lerp(_hold.HandVelocity, val, num);
				_hold.LastHandPos = position;
			}
		}
	}

	private void StartThrow(PlayerController local, Vector3 velocity)
	{
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		if (!(((Vector3)(ref velocity)).sqrMagnitude <= 1E-06f))
		{
			_throwActive = true;
			_throwVelocity = velocity;
			_throwTimeLeft = 0.25f;
			ApplyThrowVelocity(local, velocity);
		}
	}

	private void ApplyPendingThrow()
	{
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Expected O, but got Unknown
		//IL_0072: Unknown result type (might be due to invalid IL or missing references)
		//IL_007d: Expected O, but got Unknown
		//IL_009a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a5: Expected O, but got Unknown
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0102: Unknown result type (might be due to invalid IL or missing references)
		//IL_0109: Unknown result type (might be due to invalid IL or missing references)
		//IL_010e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		if (!_throwActive || _throwTimeLeft <= 0f)
		{
			return;
		}
		PlayerController current = PlayerController.Current;
		PlayerController val = ((current is SmoothLocomotionPlayerController) ? current : null);
		SmoothLocomotion val2 = (((Object)(object)val != (Object)null) ? ((SmoothLocomotionPlayerController)val).SmoothLocomotion : null);
		if ((Object)val2 == (Object)null)
		{
			_throwActive = false;
			return;
		}
		float deltaTime = Time.deltaTime;
		if (!(deltaTime <= 0f))
		{
			CharacterController characterController = val2.CharacterController;
			if ((Object)characterController != (Object)null)
			{
				characterController.Move(_throwVelocity * deltaTime);
			}
			else if ((Object)PlayerController.Current != (Object)null)
			{
				Transform transform = ((Component)PlayerController.Current).transform;
				transform.position += _throwVelocity * deltaTime;
			}
			val2.velocity = _throwVelocity;
			_throwTimeLeft -= deltaTime;
			float num = 1f - Mathf.Exp(-8f * deltaTime);
			_throwVelocity = Vector3.Lerp(_throwVelocity, Vector3.zero, num);
			if (_throwTimeLeft <= 0f || ((Vector3)(ref _throwVelocity)).sqrMagnitude <= 0.0001f)
			{
				_throwActive = false;
			}
		}
	}

	private void ApplyThrowVelocity(PlayerController local, Vector3 velocity)
	{
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Expected O, but got Unknown
		//IL_0034: Unknown result type (might be due to invalid IL or missing references)
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Expected O, but got Unknown
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0056: Unknown result type (might be due to invalid IL or missing references)
		//IL_005b: Unknown result type (might be due to invalid IL or missing references)
		PlayerController val = ((local is SmoothLocomotionPlayerController) ? local : null);
		SmoothLocomotion val2 = (((Object)(object)val != (Object)null) ? ((SmoothLocomotionPlayerController)val).SmoothLocomotion : null);
		if (!((Object)val2 == (Object)null))
		{
			val2.velocity = velocity;
			CharacterController characterController = val2.CharacterController;
			if ((Object)characterController != (Object)null)
			{
				characterController.Move(velocity * Time.deltaTime);
			}
		}
	}

	private static T GetPrivateField<T>(object target, string fieldName)
	{
		if (target == null)
		{
			return default(T);
		}
		FieldInfo field = target.GetType().GetField(fieldName, BindingFlags.Instance | BindingFlags.NonPublic);
		if (field == null)
		{
			return default(T);
		}
		return (T)field.GetValue(target);
	}

	private static float GetGripAxis(HandGestureController gc)
	{
		return GetPrivateField<float>(gc, "gripAxis");
	}

	private static bool GetIsFreeGestureMode(HandGestureController gc)
	{
		return GetPrivateField<bool>(gc, "isFreeGestureMode");
	}

	private static float[] GetSkeletonInput(HandGestureController gc)
	{
		return GetPrivateField<float[]>(gc, "skeletonInput");
	}

	private static bool TryGetHandGrip(Transform playerRoot, bool left, out HandGrip grip)
	{
		//IL_0046: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Expected O, but got Unknown
		grip = null;
		if ((Object)(object)playerRoot == (Object)null)
		{
			return false;
		}
		string needle = (left ? LeftHandBone : RightHandBone);
		Transform val = FindChildByNameFast(playerRoot, needle);
		if ((Object)(object)val == (Object)null)
		{
			return false;
		}
		grip = ((Component)val).GetComponent<HandGrip>() ?? ((Component)val).GetComponentInChildren<HandGrip>(true);
		return (Object)grip != (Object)null;
	}

	private static bool TryGetHandAndGC(Transform playerRoot, bool left, out Hand hand, out HandGestureController gc)
	{
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_002d: Expected O, but got Unknown
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Invalid comparison between Unknown and I4
		hand = null;
		gc = null;
		if ((Object)(object)playerRoot == (Object)null)
		{
			return false;
		}
		Hand[] componentsInChildren = ((Component)playerRoot).GetComponentsInChildren<Hand>(true);
		foreach (Hand val in componentsInChildren)
		{
			if ((Object)val != (Object)null && ((left && (int)val.HandIndex == 0) || (!left && (int)val.HandIndex == 1)))
			{
				hand = val;
				gc = val.GestureController;
				return true;
			}
		}
		return false;
	}

	private static bool IsSideGripping(Transform playerRoot, bool left)
	{
		if (TryGetHandGrip(playerRoot, left, out var grip) && grip.IsGripping)
		{
			return true;
		}
		if (TryGetHandAndGC(playerRoot, left, out var _, out var gc))
		{
			if (!GetIsFreeGestureMode(gc))
			{
				return GetGripAxis(gc) >= 0.6f;
			}
			float[] skeletonInput = GetSkeletonInput(gc);
			if (skeletonInput != null && skeletonInput.Length >= 5)
			{
				return (skeletonInput[1] + skeletonInput[2] + skeletonInput[3] + skeletonInput[4]) * 0.25f >= 0.8f;
			}
		}
		return false;
	}

	private static GripFlags GetRemoteGripState(Transform playerRoot)
	{
		GripFlags gripFlags = GripFlags.None;
		if (IsSideGripping(playerRoot, left: true))
		{
			gripFlags |= GripFlags.Left;
		}
		if (IsSideGripping(playerRoot, left: false))
		{
			gripFlags |= GripFlags.Right;
		}
		return gripFlags;
	}

	private static Transform FindChildByNameFast(Transform parent, string needle)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)parent == (Object)null || string.IsNullOrEmpty(needle))
		{
			return null;
		}
		Stack<Transform> stack = new Stack<Transform>();
		stack.Push(parent);
		while (stack.Count > 0)
		{
			Transform val = stack.Pop();
			if (((Object)val).name.IndexOf(needle, StringComparison.OrdinalIgnoreCase) >= 0)
			{
				return val;
			}
			for (int i = 0; i < val.childCount; i++)
			{
				stack.Push(val.GetChild(i));
			}
		}
		return null;
	}

	public void StartESP()
	{
		if (string.IsNullOrEmpty(espInput))
		{
			MelonLogger.Msg("[ESP] No input set. Type a search string first.");
			return;
		}
		espEnabled = true;
		ClearAllESPMarkers();
		EnsureESPOverlayMaterial();
		BuildInitial();
		_nextRefreshTime = Time.time + espRefreshInterval;
		MelonLogger.Msg($"[ESP] Started (auto-refresh every {espRefreshInterval:0.0}s) for '{espInput}'");
	}

	public void StopESP()
	{
		espEnabled = false;
		ClearAllESPMarkers();
		MelonLogger.Msg("[ESP] Stopped and cleared.");
	}

	private void ESP_Tick()
	{
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_002e: Expected O, but got Unknown
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Expected O, but got Unknown
		//IL_0091: Unknown result type (might be due to invalid IL or missing references)
		//IL_009b: Expected O, but got Unknown
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bb: Expected O, but got Unknown
		//IL_0147: Unknown result type (might be due to invalid IL or missing references)
		//IL_014e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0193: Unknown result type (might be due to invalid IL or missing references)
		//IL_0198: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01da: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_0201: Unknown result type (might be due to invalid IL or missing references)
		//IL_0243: Unknown result type (might be due to invalid IL or missing references)
		//IL_024a: Unknown result type (might be due to invalid IL or missing references)
		if (!espEnabled)
		{
			return;
		}
		PlayerController current = PlayerController.Current;
		Camera val = (((Object)(object)current != (Object)null) ? current.Camera : null);
		if ((Object)val == (Object)null)
		{
			return;
		}
		float time = Time.time;
		for (int num = _markers.Count - 1; num >= 0; num--)
		{
			MarkerInfo markerInfo = _markers[num];
			if ((Object)markerInfo.target == (Object)null || !IsLiveTransform(markerInfo.target))
			{
				if ((Object)(object)markerInfo.labelRoot != (Object)null)
				{
					Object.Destroy((Object)markerInfo.labelRoot);
				}
				if ((Object)(object)markerInfo.ball != (Object)null)
				{
					Object.Destroy((Object)markerInfo.ball);
				}
				_markers.RemoveAt(num);
			}
			else
			{
				if (!markerInfo.parented)
				{
					markerInfo.ball.transform.position = markerInfo.target.position;
					if (espCopyRotation)
					{
						markerInfo.ball.transform.rotation = markerInfo.target.rotation;
					}
				}
				float num2 = 1f + Mathf.Sin((time + markerInfo.phaseJitter) * 3.2f) * 0.15f;
				markerInfo.ball.transform.localScale = markerInfo.baseScale * num2;
				if (espShowLabels && (Object)(object)markerInfo.labelRoot != (Object)null && (Object)(object)markerInfo.labelTM != (Object)null)
				{
					Vector3 val2 = markerInfo.ball.transform.position + Vector3.up * espLabelYOffset;
					markerInfo.labelRoot.transform.position = val2;
					markerInfo.labelRoot.transform.rotation = Quaternion.LookRotation(markerInfo.labelRoot.transform.position - ((Component)val).transform.position);
					float num3 = Mathf.Max(Vector3.Distance(val2, ((Component)val).transform.position), 1f);
					float num4 = espLabelBaseSize * Mathf.Clamp(num3 * 0.06f, 0.5f, 4f);
					markerInfo.labelRoot.transform.localScale = Vector3.one * num4;
				}
			}
		}
		if (Time.time >= _nextRefreshTime)
		{
			TryAutoRefreshAddOnly();
			_nextRefreshTime = Time.time + espRefreshInterval;
		}
	}

	private void BuildInitial()
	{
		List<Transform> list = FindTransformsByNameContains(espInput);
		if (list == null || list.Count == 0)
		{
			MelonLogger.Msg("[ESP] No objects found containing '" + espInput + "'");
			return;
		}
		int num = 0;
		for (int i = 0; i < list.Count; i++)
		{
			if (num >= maxESPMarkers)
			{
				break;
			}
			Transform val = list[i];
			if ((Object)(object)val != (Object)null)
			{
				MarkerInfo markerInfo = CreateMarkerForTarget(val);
				if (markerInfo != null)
				{
					_markers.Add(markerInfo);
					espMarkers.Add(markerInfo.ball);
					num++;
				}
			}
		}
		if (list.Count > maxESPMarkers)
		{
			MelonLogger.Msg($"[ESP] WARNING: {list.Count - maxESPMarkers} more matches skipped due to cap");
		}
		MelonLogger.Msg($"[ESP] created {num} marker(s)");
	}

	private void TryAutoRefreshAddOnly()
	{
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Expected O, but got Unknown
		if (string.IsNullOrEmpty(espInput))
		{
			return;
		}
		HashSet<int> hashSet = new HashSet<int>();
		foreach (MarkerInfo marker in _markers)
		{
			if ((Object)(marker?.target) != (Object)null && IsLiveTransform(marker.target))
			{
				hashSet.Add(GetDedupKey(marker.target));
			}
		}
		List<Transform> list = FindTransformsByNameContains(espInput);
		int num = 0;
		for (int i = 0; i < list.Count; i++)
		{
			if (_markers.Count >= maxESPMarkers)
			{
				break;
			}
			Transform val = list[i];
			if ((Object)(object)val == (Object)null || !IsLiveTransform(val))
			{
				continue;
			}
			int dedupKey = GetDedupKey(val);
			if (!hashSet.Contains(dedupKey))
			{
				MarkerInfo markerInfo = CreateMarkerForTarget(val);
				if (markerInfo != null)
				{
					_markers.Add(markerInfo);
					espMarkers.Add(markerInfo.ball);
					hashSet.Add(dedupKey);
					num++;
				}
			}
		}
		if (num > 0)
		{
			MelonLogger.Msg($"[ESP] auto-refresh: added {num} new marker(s) (total {_markers.Count})");
		}
	}

	public void ClearAllESPMarkers()
	{
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Expected O, but got Unknown
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Expected O, but got Unknown
		for (int num = _markers.Count - 1; num >= 0; num--)
		{
			MarkerInfo markerInfo = _markers[num];
			if ((Object)(object)markerInfo.labelRoot != (Object)null)
			{
				Object.Destroy((Object)markerInfo.labelRoot);
			}
			if ((Object)(object)markerInfo.ball != (Object)null)
			{
				Object.Destroy((Object)markerInfo.ball);
			}
		}
		_markers.Clear();
		espMarkers.Clear();
	}

	private MarkerInfo CreateMarkerForTarget(Transform target)
	{
		//IL_021b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0226: Expected O, but got Unknown
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ed: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Expected O, but got Unknown
		//IL_009d: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Expected O, but got Unknown
		//IL_00de: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e4: Expected O, but got Unknown
		//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ef: Expected O, but got Unknown
		//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_0106: Unknown result type (might be due to invalid IL or missing references)
		//IL_010b: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_011f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0178: Unknown result type (might be due to invalid IL or missing references)
		//IL_019c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a7: Expected O, but got Unknown
		try
		{
			GameObject val = GameObject.CreatePrimitive((PrimitiveType)0);
			((Object)val).name = "ESP_Marker_" + ((Object)target).name;
			if (espParentToTarget)
			{
				val.transform.SetParent(target, true);
			}
			val.transform.position = target.position;
			val.transform.localScale = Vector3.one * espWorldSize;
			Collider component = val.GetComponent<Collider>();
			if ((Object)(object)component != (Object)null)
			{
				Object.Destroy((Object)component);
			}
			Renderer component2 = val.GetComponent<Renderer>();
			if ((Object)(object)component2 != (Object)null)
			{
				component2.shadowCastingMode = (ShadowCastingMode)0;
				component2.receiveShadows = false;
				if ((Object)espOverlayMaterial == (Object)null)
				{
					EnsureESPOverlayMaterial();
				}
				component2.sharedMaterial = espOverlayMaterial;
				((Object)component2.sharedMaterial).hideFlags = (HideFlags)61;
			}
			GameObject val2 = null;
			TextMesh val3 = null;
			if (espShowLabels)
			{
				val2 = new GameObject("ESP_LabelRoot");
				Object.DontDestroyOnLoad((Object)val2);
				val2.transform.position = target.position + Vector3.up * espLabelYOffset;
				GameObject val4 = new GameObject("ESP_Label");
				val4.transform.SetParent(val2.transform, false);
				val3 = val4.AddComponent<TextMesh>();
				val3.text = TruncateName(((Object)target).name, espLabelMaxLen);
				val3.fontSize = 64;
				val3.characterSize = 0.02f;
				val3.anchor = (TextAnchor)4;
				val3.alignment = (TextAlignment)1;
				val3.color = Color.white;
				MeshRenderer component3 = ((Component)val3).GetComponent<MeshRenderer>();
				if ((Object)(object)component3 != (Object)null && (Object)((Renderer)component3).sharedMaterial != (Object)null)
				{
					try
					{
						((Renderer)component3).sharedMaterial.renderQueue = 5000;
					}
					catch
					{
					}
				}
			}
			return new MarkerInfo
			{
				target = target,
				ball = val,
				labelRoot = val2,
				labelTM = val3,
				baseScale = val.transform.localScale,
				phaseJitter = Random.value * 10f,
				parented = espParentToTarget
			};
		}
		catch (Exception arg)
		{
			MelonLogger.Msg(string.Format("[ESP] failed to create marker for {0}: {1}", ((Object)target != (Object)null) ? ((Object)target).name : "null", arg));
			return null;
		}
	}

	private void EnsureESPOverlayMaterial()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_002d: Expected O, but got Unknown
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Expected O, but got Unknown
		//IL_006c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ab: Expected O, but got Unknown
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		if (!((Object)espOverlayMaterial != (Object)null))
		{
			Shader val = Shader.Find("Hidden/Internal-Colored");
			if ((Object)val != (Object)null)
			{
				espOverlayMaterial = new Material(val);
				TrySetInt(espOverlayMaterial, "_ZWrite", 0);
				TrySetInt(espOverlayMaterial, "_ZTest", 8);
				TrySetColor(espOverlayMaterial, "_Color", espColor);
				espOverlayMaterial.renderQueue = 5000;
			}
			else
			{
				espOverlayMaterial = new Material(Shader.Find("Unlit/Color") ?? Shader.Find("Standard"));
				TrySetColor(espOverlayMaterial, "_Color", espColor);
				TrySetInt(espOverlayMaterial, "_ZWrite", 0);
				TrySetInt(espOverlayMaterial, "_ZTest", 8);
				espOverlayMaterial.renderQueue = 5000;
			}
			((Object)espOverlayMaterial).hideFlags = (HideFlags)61;
		}
	}

	private void TrySetInt(Material m, string name, int v)
	{
		if (m.HasProperty(name))
		{
			m.SetInt(name, v);
		}
	}

	private void TrySetColor(Material m, string name, Color c)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		if (m.HasProperty(name))
		{
			m.SetColor(name, c);
			return;
		}
		try
		{
			m.color = c;
		}
		catch
		{
		}
	}

	private int GetDedupKey(Transform t)
	{
		if (!espOnePerRoot)
		{
			return ((Object)t).GetInstanceID();
		}
		return ((Object)t.root).GetInstanceID();
	}

	private bool IsLiveTransform(Transform t)
	{
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)(object)t == (Object)null)
		{
			return false;
		}
		GameObject gameObject = ((Component)t).gameObject;
		Scene scene = gameObject.scene;
		if (!((Scene)(ref scene)).IsValid())
		{
			scene = gameObject.scene;
			return ((Scene)(ref scene)).name == null;
		}
		return true;
	}

	private List<Transform> FindTransformsByNameContains(string name)
	{
		List<Transform> list = new List<Transform>();
		if (string.IsNullOrEmpty(name))
		{
			return list;
		}
		IEnumerable<Transform> enumerable = EnumerateAllTransformsSafely();
		HashSet<int> hashSet = new HashSet<int>();
		foreach (Transform item in enumerable)
		{
			if (FilterTransform(item) && ((Object)item).name.IndexOf(name, StringComparison.OrdinalIgnoreCase) >= 0)
			{
				int dedupKey = GetDedupKey(item);
				if (!espOnePerRoot || !hashSet.Contains(dedupKey))
				{
					hashSet.Add(dedupKey);
					list.Add(item);
				}
			}
		}
		return list;
	}

	private IEnumerable<Transform> EnumerateAllTransformsSafely()
	{
		try
		{
			return Object.FindObjectsOfType<Transform>(true);
		}
		catch
		{
			return Resources.FindObjectsOfTypeAll<Transform>().Where(delegate(Transform t)
			{
				//IL_0013: Unknown result type (might be due to invalid IL or missing references)
				//IL_0025: Unknown result type (might be due to invalid IL or missing references)
				//IL_002a: Unknown result type (might be due to invalid IL or missing references)
				//IL_001b: Unknown result type (might be due to invalid IL or missing references)
				//IL_0022: Invalid comparison between Unknown and I4
				//IL_0035: Unknown result type (might be due to invalid IL or missing references)
				//IL_003a: Unknown result type (might be due to invalid IL or missing references)
				if ((Object)(object)t == (Object)null)
				{
					return false;
				}
				GameObject gameObject = ((Component)t).gameObject;
				if ((int)((Object)gameObject).hideFlags == 0 || (int)((Object)gameObject).hideFlags == 61)
				{
					Scene scene = gameObject.scene;
					if (!((Scene)(ref scene)).IsValid())
					{
						scene = gameObject.scene;
						return ((Scene)(ref scene)).name == null;
					}
					return true;
				}
				return false;
			});
		}
	}

	private string TruncateName(string s, int max)
	{
		if (string.IsNullOrEmpty(s))
		{
			return s;
		}
		if (s.Length <= max)
		{
			return s;
		}
		return s.Substring(0, Mathf.Max(0, max - 1)) + "…";
	}

	private bool FilterTransform(Transform t)
	{
		if ((Object)(object)t == (Object)null || !IsLiveTransform(t))
		{
			return false;
		}
		string name = ((Object)t).name;
		if (!string.IsNullOrEmpty(name) && !name.StartsWith("Bone", StringComparison.OrdinalIgnoreCase))
		{
			return name.IndexOf("Armature", StringComparison.OrdinalIgnoreCase) < 0;
		}
		return false;
	}

	public void ToggleSpinning()
	{
		//IL_019e: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a9: Expected O, but got Unknown
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Expected O, but got Unknown
		//IL_01d0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01db: Expected O, but got Unknown
		//IL_01b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c0: Expected O, but got Unknown
		//IL_0112: Unknown result type (might be due to invalid IL or missing references)
		//IL_011d: Expected O, but got Unknown
		//IL_0125: Unknown result type (might be due to invalid IL or missing references)
		//IL_013f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0149: Expected O, but got Unknown
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Expected O, but got Unknown
		//IL_015d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0168: Expected O, but got Unknown
		//IL_0177: Unknown result type (might be due to invalid IL or missing references)
		//IL_0182: Expected O, but got Unknown
		//IL_00a2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a9: Expected O, but got Unknown
		//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b6: Expected O, but got Unknown
		//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Expected O, but got Unknown
		//IL_00bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0102: Unknown result type (might be due to invalid IL or missing references)
		//IL_0107: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
		_isSpinning = !_isSpinning;
		PlayerController current = PlayerController.Current;
		if (_isSpinning)
		{
			if ((Object)current != (Object)null)
			{
				Controller leftController = current.LeftController;
				object obj;
				if ((Object)(object)leftController == (Object)null)
				{
					obj = null;
				}
				else
				{
					Hand hand = leftController.Hand;
					obj = (((Object)(object)hand != (Object)null) ? ((Component)hand).transform : null);
				}
				Transform val = (Transform)obj;
				Controller rightController = current.RightController;
				object obj2;
				if ((Object)(object)rightController == (Object)null)
				{
					obj2 = null;
				}
				else
				{
					Hand hand2 = rightController.Hand;
					obj2 = (((Object)(object)hand2 != (Object)null) ? ((Component)hand2).transform : null);
				}
				Transform val2 = (Transform)obj2;
				if ((Object)val != (Object)null)
				{
					_lockedLeftHandPos = val.position;
					_lockedLeftHandRot = val.rotation;
				}
				if ((Object)val2 != (Object)null)
				{
					_lockedRightHandPos = val2.position;
					_lockedRightHandRot = val2.rotation;
				}
				_lockedBodyRotation = ((Component)current).transform.rotation;
			}
			if ((Object)_spinningCamera == (Object)null)
			{
				_spinningCamera = new GameObject("SpinningCamera").AddComponent<Camera>();
				Object.DontDestroyOnLoad((Object)((Component)_spinningCamera).gameObject);
			}
			Camera val3 = (((Object)(object)current != (Object)null) ? current.Camera : null);
			if ((Object)val3 != (Object)null)
			{
				((Behaviour)val3).enabled = false;
			}
			if ((Object)thirdPersonCam != (Object)null)
			{
				((Behaviour)thirdPersonCam).enabled = false;
			}
			((Behaviour)_spinningCamera).enabled = true;
			return;
		}
		if ((Object)current != (Object)null)
		{
			Camera camera = current.Camera;
			if ((Object)camera != (Object)null)
			{
				((Behaviour)camera).enabled = true;
			}
		}
		if ((Object)_spinningCamera != (Object)null)
		{
			((Behaviour)_spinningCamera).enabled = false;
		}
	}

	private void Spinning_Tick()
	{
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		//IL_001b: Expected O, but got Unknown
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_006d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0073: Expected O, but got Unknown
		//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Expected O, but got Unknown
		//IL_00b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Expected O, but got Unknown
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e5: Expected O, but got Unknown
		//IL_00c2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_0108: Unknown result type (might be due to invalid IL or missing references)
		//IL_0113: Unknown result type (might be due to invalid IL or missing references)
		//IL_011d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0122: Unknown result type (might be due to invalid IL or missing references)
		//IL_0127: Unknown result type (might be due to invalid IL or missing references)
		//IL_0137: Unknown result type (might be due to invalid IL or missing references)
		//IL_013c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ab: Expected O, but got Unknown
		//IL_014e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0156: Unknown result type (might be due to invalid IL or missing references)
		//IL_0162: Unknown result type (might be due to invalid IL or missing references)
		//IL_0175: Unknown result type (might be due to invalid IL or missing references)
		//IL_017a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0181: Unknown result type (might be due to invalid IL or missing references)
		//IL_018b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01da: Unknown result type (might be due to invalid IL or missing references)
		//IL_01df: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f9: Unknown result type (might be due to invalid IL or missing references)
		if (!_isSpinning)
		{
			return;
		}
		PlayerController current = PlayerController.Current;
		if (!((Object)current == (Object)null))
		{
			((Component)current).transform.Rotate(Vector3.up, 100f * Time.deltaTime);
			Controller leftController = current.LeftController;
			object obj;
			if ((Object)(object)leftController == (Object)null)
			{
				obj = null;
			}
			else
			{
				Hand hand = leftController.Hand;
				obj = (((Object)(object)hand != (Object)null) ? ((Component)hand).transform : null);
			}
			Transform val = (Transform)obj;
			Controller rightController = current.RightController;
			object obj2;
			if ((Object)(object)rightController == (Object)null)
			{
				obj2 = null;
			}
			else
			{
				Hand hand2 = rightController.Hand;
				obj2 = (((Object)(object)hand2 != (Object)null) ? ((Component)hand2).transform : null);
			}
			Transform val2 = (Transform)obj2;
			if ((Object)val != (Object)null)
			{
				val.position = _lockedLeftHandPos;
				val.rotation = _lockedLeftHandRot;
			}
			if ((Object)val2 != (Object)null)
			{
				val2.position = _lockedRightHandPos;
				val2.rotation = _lockedRightHandRot;
			}
			((Component)current).transform.rotation = _lockedBodyRotation * Quaternion.AngleAxis(((Component)current).transform.eulerAngles.y, Vector3.up);
			Vector2 val3 = _xrLStick.ReadValue<Vector2>();
			if (((Vector2)(ref val3)).sqrMagnitude > 0.01f)
			{
				Vector3 val4 = default(Vector3);
				((Vector3)(ref val4))._002Ector(val3.x, 0f, val3.y);
				Transform transform = ((Component)current).transform;
				transform.position += val4 * 5f * Time.deltaTime;
			}
			if ((Object)_spinningCamera != (Object)null && ((Behaviour)_spinningCamera).enabled)
			{
				((Component)_spinningCamera).transform.position = ((Component)current).transform.position + Vector3.up * 10f;
				((Component)_spinningCamera).transform.rotation = Quaternion.LookRotation(Vector3.down);
			}
		}
	}

	private void HandleReactorSweepHotkey()
	{
	}

	private static void SetLayerRecursively(GameObject go, int layer)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)go == (Object)null)
		{
			return;
		}
		go.layer = layer;
		foreach (Transform item in go.transform)
		{
			SetLayerRecursively(((Component)item).gameObject, layer);
		}
	}

	public void SetControlActive(bool active)
	{
		if (_controlActive == active)
		{
			return;
		}
		_controlActive = active;
		try
		{
			if (active)
			{
				Actions.take_control();
			}
			else
			{
				Actions.exit_control();
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Failed to toggle orb control: " + ex.GetBaseException().Message);
		}
	}

	public void DisconnectToMenu()
	{
		try
		{
			Basemodal.main_values.pressed_disconnect = true;
			SendConsoleCmd("disconnect");
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Disconnect failed: " + ex.GetBaseException().Message);
		}
	}

	public void SetMinimapVisible(bool show)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		if ((Object)boardClone == (Object)null)
		{
			if (!show)
			{
				minimapVisible = false;
				return;
			}
			ToggleMiniMap();
			if (!minimapVisible)
			{
				ToggleMiniMap();
			}
		}
		else if (minimapVisible != show)
		{
			ToggleMiniMap();
			if (minimapVisible != show)
			{
				ToggleMiniMap();
			}
		}
	}

	public void SetAutoHarvestEnabled(bool enable)
	{
		if (_autoHarvestEnabled != enable)
		{
			_autoHarvestEnabled = enable;
			Basemodal.main_values.is_harvesting = enable;
			_autoHarvestTimer = 0f;
			_autoHarvestResetTimer = 0f;
			if (!enable)
			{
				Actions.ClearImpactCache();
			}
		}
	}

	public void SetAutoReviveEnabled(bool enable)
	{
		if (_autoReviveEnabled != enable)
		{
			_autoReviveEnabled = enable;
			_autoReviveInProgress = false;
			_autoReviveNextTime = 0f;
		}
	}

	public static void TeleportLocalPlayer(Vector3 worldPosition, LocomotionFunction locomotion)
	{
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Expected O, but got Unknown
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		PlayerController current = PlayerController.Current;
		LocomotionController val = (((Object)(object)current != (Object)null) ? current.LocomotionController : null);
		if ((Object)val == (Object)null)
		{
			return;
		}
		try
		{
			val.MoveTo(worldPosition, locomotion);
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Teleport failed: " + ex.GetBaseException().Message);
		}
	}

	private Texture2D MakeTex(int w, int h, Color c)
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0008: Expected O, but got Unknown
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		Texture2D val = new Texture2D(w, h);
		Color[] array = (Color[])(object)new Color[w * h];
		for (int i = 0; i < array.Length; i++)
		{
			array[i] = c;
		}
		val.SetPixels(array);
		val.Apply();
		return val;
	}

	private float PFV3_FloatFieldInline(float v, float width)
	{
		float.TryParse(GUILayout.TextField(v.ToString("0.###", CultureInfo.InvariantCulture), (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(width) }), NumberStyles.Float, CultureInfo.InvariantCulture, out v);
		return v;
	}

	public override void OnApplicationStart()
	{
		//IL_0005: Unknown result type (might be due to invalid IL or missing references)
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		Harmony val = new Harmony("Idknameyet.RestrictedUnifiedMod");
		val.PatchAll(typeof(AdonaiUnifiedMod));
		val.PatchAll(typeof(DLLPatches));
		val.PatchAll(typeof(Pc2qPatch));
	}

public override void OnApplicationQuit()
{
    // Do nothing
    MelonLogger.Msg("Prevented application from quitting.");
}

	private void RunAutoHarvest()
	{
		//IL_0015: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Expected O, but got Unknown
		if (!_autoHarvestEnabled || !_sceneReady || (Object)PlayerController.Current == (Object)null)
		{
			return;
		}
		_autoHarvestTimer += Time.deltaTime;
		_autoHarvestResetTimer += Time.deltaTime;
		if (_autoHarvestTimer >= 0f)
		{
			_autoHarvestTimer = 0f;
			try
			{
				Actions.AutoHarvestTick();
			}
			catch (Exception ex)
			{
				MelonLogger.Warning("[Unified] Auto-harvest tick failed: " + ex.GetBaseException().Message);
			}
		}
		if (Actions.ImpactList.Count == 0)
		{
			_autoHarvestResetTimer = 0f;
		}
		if (_autoHarvestResetTimer >= 30f)
		{
			_autoHarvestResetTimer = 0f;
			Actions.ClearImpactCache();
		}
	}

	private void RunAutoRevive()
	{
		if (_autoReviveEnabled && _sceneReady && !_autoReviveInProgress && !(Time.realtimeSinceStartup < _autoReviveNextTime) && IsPlayerDowned())
		{
			_autoReviveInProgress = true;
			_autoReviveNextTime = Time.realtimeSinceStartup + 2f;
			MelonCoroutines.Start(AutoReviveConsumeOrbsCo());
		}
	}

	private IEnumerator AutoReviveConsumeOrbsCo()
	{
		try
		{
			while (IsPlayerDowned())
			{
				Transform head = Actions.find_head();
				if ((Object)head != (Object)null)
				{
					Actions.move_world_hand("right", head.position);
					while (IsPlayerDowned())
					{
						Transform val = Actions.find_hand(right: true);
						if ((Object)val == (Object)null)
						{
							break;
						}
						Vector3 val2 = val.position - head.position;
						if (!(((Vector3)(ref val2)).magnitude < 0.02f))
						{
							break;
						}
						Actions.move_world_hand("right", head.position);
						yield return (object)new WaitForSeconds(0.1f);
					}
				}
				if (!IsPlayerDowned())
				{
					break;
				}
				ReviveOrb[] array = Object.FindObjectsOfType<ReviveOrb>();
				for (int i = 0; i < array.Length; i++)
				{
					if (!IsPlayerDowned())
					{
						break;
					}
					ReviveOrb val3 = array[i];
					if ((Object)val3 == (Object)null)
					{
						continue;
					}
					Pickup component = ((Component)((Component)val3).transform).GetComponent<Pickup>();
					if ((Object)component == (Object)null)
					{
						continue;
					}
					((Behaviour)component).enabled = true;
					if (!((Object)((Component)val3).GetComponent<Interactable>() == (Object)null))
					{
						PlayerController current = PlayerController.Current;
						if (!((Object)current == (Object)null))
						{
							current.RightController.Interactor.StartInteract((Interactable)component, false, false, false);
							current.RightController.Interactor.StopInteract(false, false);
							Actions.grab_item(right: true);
							yield return (object)new WaitForSeconds(0.02f);
							Actions.drop_item(right: true);
							yield return (object)new WaitForSeconds(0.02f);
						}
					}
				}
				yield return (object)new WaitForSeconds(0.2f);
			}
		}
		finally
		{
			AdonaiUnifiedMod adonaiUnifiedMod = this;
			adonaiUnifiedMod._autoReviveInProgress = false;
			adonaiUnifiedMod._autoReviveNextTime = Time.realtimeSinceStartup + 1f;
		}
	}

	private bool IsPlayerDowned()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Expected O, but got Unknown
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Expected O, but got Unknown
		PlayerController current = PlayerController.Current;
		if ((Object)current == (Object)null)
		{
			return false;
		}
		try
		{
			Type type = AccessTools.TypeByName("Township.Downed.DownedCharacter") ?? AccessTools.TypeByName("DownedCharacter");
			if (type != null)
			{
				Component component = ((Component)current).GetComponent(type);
				if ((Object)component != (Object)null)
				{
					PropertyInfo property = type.GetProperty("IsDowned", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
					if (property != null && property.PropertyType == typeof(bool))
					{
						return (bool)property.GetValue(component);
					}
					FieldInfo fieldInfo = type.GetField("IsDowned", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic) ?? type.GetField("isDowned", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic) ?? type.GetField("_isDowned", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
					if (fieldInfo != null && fieldInfo.FieldType == typeof(bool))
					{
						return (bool)fieldInfo.GetValue(component);
					}
				}
			}
			PropertyInfo property2 = ((object)current).GetType().GetProperty("PlayerState", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
			if (property2 != null)
			{
				object value = property2.GetValue(current);
				if (value != null)
				{
					string text = value.ToString();
					if (!string.IsNullOrEmpty(text) && text.IndexOf("Downed", StringComparison.OrdinalIgnoreCase) >= 0)
					{
						return true;
					}
				}
			}
		}
		catch
		{
		}
		return false;
	}

	public override void OnFixedUpdate()
	{
	}

	public override void OnUpdate()
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Expected O, but got Unknown
		if (!CSRuntime.IsAlive)
		{
			Transform val = Actions.find_hand(right: true);
			if ((Object)val != (Object)null)
			{
				CSRuntime.SpawnForHand(val);
			}
		}
		_csEnabled = true;
		_smiteEnabled = true;
		_bladeHeldFxEnabled = true;
		CSRuntime.SetSmiteEnabled(enabled: true);
		CSRuntime.SetBladeHeldFxEnabled(enabled: true);
		RunAutoRevive();
	}

	private void EnsureApiServerRegistration()
	{
		if (_sceneReady && _api != null && _api.IsAuthenticated && _api.is_alive)
		{
			int activeServerId = GetActiveServerId();
			if (activeServerId != 0 && (!_apiRegisteredWithServer || _apiRegisteredServerId != activeServerId) && (_apiJoinTask == null || _apiJoinTask.IsCompleted))
			{
				_apiJoinTask = RegisterApiForServerAsync(activeServerId);
			}
		}
	}

	private int GetActiveServerId()
	{
		if (Basemodal.main_values.target_server_id != 0)
		{
			return Basemodal.main_values.target_server_id;
		}
		if (Basemodal.main_values.connected_serverid != 0)
		{
			return Basemodal.main_values.connected_serverid;
		}
		if (int.TryParse(_srvId, out var result) && result > 0)
		{
			return result;
		}
		return 0;
	}

	private async Task RegisterApiForServerAsync(int serverId)
	{
		_ = 2;
		try
		{
			_apiRegisteredWithServer = true;
			_apiRegisteredServerId = serverId;
			Basemodal.main_values.target_server_id = serverId;
			await _api.send_to_ws($"event joined-server {serverId}");
			await _api.SendCommandAsync("player list network");
			await _api.ConnectPhysicsServer(serverId);
			SpawnOwnOrbAsync();
		}
		catch (Exception ex)
		{
			_apiRegisteredWithServer = false;
			MelonLogger.Warning("[API] server registration failed: " + ex.GetBaseException().Message);
		}
	}

	private async Task SpawnOwnOrbAsync()
	{
		_ = 1;
		try
		{
			UserData userData = await _api.GetMe();
			if (userData == null || string.IsNullOrWhiteSpace(userData.username))
			{
				return;
			}
			OrbData orb = await _api.GetOrb(userData.username);
			if (orb == null)
			{
				return;
			}
			IHighLevelApiClient apiClient = ApiAccess.ApiClient;
			int? num;
			if (apiClient == null)
			{
				num = null;
			}
			else
			{
				IUserApiClient userClient = apiClient.UserClient;
				if (userClient == null)
				{
					num = null;
				}
				else
				{
					UserInfoWithPermissions loggedInUserInfo = userClient.LoggedInUserInfo;
					num = ((loggedInUserInfo != null) ? new int?(((UserInfo)loggedInUserInfo).Identifier) : ((int?)null));
				}
			}
			int? num2 = num;
			int ownerId = num2.GetValueOrDefault();
			if (ownerId != 0)
			{
				UnityMainThreadDispatcher.Enqueue(delegate
				{
					OrbUtils.spawn_orb(orb, ownerId);
				});
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[API] spawn orb failed: " + ex.GetBaseException().Message);
		}
	}

	private void PollPlayersForApi()
	{
		if (!_sceneReady || _api == null || !_api.IsAuthenticated || !_api.is_alive || Time.realtimeSinceStartup < _playerPollNextTime)
		{
			return;
		}
		_playerPollNextTime = Time.realtimeSinceStartup + 1f;
		List<IPlayer> list = Player.AllPlayers?.ToList();
		if (list == null)
		{
			return;
		}
		HashSet<int> currentIds = new HashSet<int>(list.Select((IPlayer p) => p.UserInfo.Identifier));
		foreach (IPlayer item in list)
		{
			if (!_knownPlayers.Contains(item.UserInfo.Identifier))
			{
				_knownPlayers.Add(item.UserInfo.Identifier);
				OnApiPlayerJoined(item);
			}
		}
		foreach (int id in _knownPlayers.Where((int item) => !currentIds.Contains(item)).ToList())
		{
			_knownPlayers.Remove(id);
			IPlayer player = ((IEnumerable<IPlayer>)list).FirstOrDefault((Func<IPlayer, bool>)((IPlayer x) => x.UserInfo.Identifier == id));
			OnApiPlayerLeft(player, id);
		}
	}

	private async Task OnApiPlayerJoined(IPlayer player)
	{
		_ = 1;
		try
		{
			int identifier = player.UserInfo.Identifier;
			IHighLevelApiClient apiClient = ApiAccess.ApiClient;
			int? num;
			if (apiClient == null)
			{
				num = null;
			}
			else
			{
				IUserApiClient userClient = apiClient.UserClient;
				if (userClient == null)
				{
					num = null;
				}
				else
				{
					UserInfoWithPermissions loggedInUserInfo = userClient.LoggedInUserInfo;
					num = ((loggedInUserInfo != null) ? new int?(((UserInfo)loggedInUserInfo).Identifier) : ((int?)null));
				}
			}
			if (identifier == num)
			{
				return;
			}
			websocket_client websocket_client = await _api.GetClient(player.UserInfo.Username);
			if (websocket_client == null)
			{
				EnqueueChat("network", player.UserInfo.Username + " joined the server.", Color.white);
				return;
			}
			string text = websocket_client.user?.account_data?.displayName ?? websocket_client.user?.username ?? player.UserInfo.Username;
			EnqueueChat("network", text + " (" + player.UserInfo.Username + ") joined the server.", Color.cyan);
			int ownerId = websocket_client.attaccount?.userid ?? player.UserInfo.Identifier;
			if (!((Object)OrbUtils.get_orb(ownerId) == (Object)null))
			{
				return;
			}
			OrbData orb = await _api.GetOrb(websocket_client.user?.username ?? player.UserInfo.Username);
			if (orb != null)
			{
				UnityMainThreadDispatcher.Enqueue(delegate
				{
					OrbUtils.spawn_orb(orb, ownerId);
				});
			}
		}
		catch (Exception ex)
		{
			object obj;
			if (player == null)
			{
				obj = null;
			}
			else
			{
				UserInfoAndRole userInfo = player.UserInfo;
				obj = ((userInfo != null) ? userInfo.Username : null);
			}
			MelonLogger.Warning("[API] failed to process join for " + (string)obj + ": " + ex.GetBaseException().Message);
		}
	}

	private void OnApiPlayerLeft(IPlayer player, int id)
	{
		//IL_00a9: Unknown result type (might be due to invalid IL or missing references)
		IHighLevelApiClient apiClient = ApiAccess.ApiClient;
		int? num;
		if (apiClient == null)
		{
			num = null;
		}
		else
		{
			IUserApiClient userClient = apiClient.UserClient;
			if (userClient == null)
			{
				num = null;
			}
			else
			{
				UserInfoWithPermissions loggedInUserInfo = userClient.LoggedInUserInfo;
				num = ((loggedInUserInfo != null) ? new int?(((UserInfo)loggedInUserInfo).Identifier) : ((int?)null));
			}
		}
		int? num2 = num;
		if (!num2.HasValue || id != num2.Value)
		{
			object obj;
			if (player == null)
			{
				obj = null;
			}
			else
			{
				UserInfoAndRole userInfo = player.UserInfo;
				obj = ((userInfo != null) ? userInfo.Username : null);
			}
			if (obj == null)
			{
				obj = id.ToString();
			}
			string text = (string)obj;
			EnqueueChat("network", text + " left the server.", Color.white);
			OrbUtils.despawn_orb(id);
			_api?.send_to_ws("player list network");
		}
	}

	private void ResetApiSessionState()
	{
		_knownPlayers.Clear();
		_apiRegisteredWithServer = false;
		_apiRegisteredServerId = -1;
		_apiJoinTask = Task.CompletedTask;
	}

	private bool PointerOverUnifiedUI()
	{
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		if (!_menuVisible)
		{
			return false;
		}
		try
		{
			Mouse current = Mouse.current;
			if (current == null)
			{
				return false;
			}
			Vector2 val = ((InputControl<Vector2>)(object)((Pointer)current).position).ReadValue();
			val.y = (float)Screen.height - val.y;
			return ((Rect)(ref _menuBounds)).Contains(val);
		}
		catch
		{
			return false;
		}
	}

	private void SceneLoaded(Scene scene, LoadSceneMode mode)
	{
		_sceneReady = ((Scene)(ref scene)).buildIndex == 4 || ((Scene)(ref scene)).buildIndex == 6;
		Basemodal.main_values.is_loaded_in = _sceneReady;
		if (!_sceneReady)
		{
			ResetApiSessionState();
			return;
		}
		_knownPlayers.Clear();
		ReapplyDebugFeatureToggles();
	}

	private void ReapplyDebugFeatureToggles()
	{
		if (_impactDebugSubscribed)
		{
			SetImpactDebugSubscribed(enabled: true);
		}
		if (_damageDebugEnabled)
		{
			SetDamageDebugEnabled(enabled: true);
		}
		if (_modMicMuteActive)
		{
			SetModMicMute(enabled: true);
		}
	}

	private void SetDebugPermissionBypass(bool enabled)
	{
		ForceDebugPermissions = enabled;
		_statusLeft = (enabled ? "Debug features unlocked" : "Debug features reset to default.");
	}

	private void SetImpactDebugSubscribed(bool enabled)
	{
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Expected O, but got Unknown
		_impactDebugSubscribed = enabled;
		try
		{
			if ((Object)ImpactSystemHitManager.Instance == (Object)null)
			{
				_statusLeft = "Hit debug manager not ready.";
				return;
			}
			ImpactSystemHitManager.SubscribeToServerHits(enabled);
			_statusLeft = (enabled ? "Subscribed to hit debug feed." : "Hit debug feed disabled.");
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Impact debug toggle failed: " + ex.GetBaseException().Message);
			_statusLeft = "Hit debug toggle failed.";
		}
	}

	private void SetDamageDebugEnabled(bool enabled)
	{
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Expected O, but got Unknown
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Expected O, but got Unknown
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Expected O, but got Unknown
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_008e: Expected O, but got Unknown
		_damageDebugEnabled = enabled;
		try
		{
			DamageDebugDisplay[] array = Resources.FindObjectsOfTypeAll<DamageDebugDisplay>();
			foreach (DamageDebugDisplay val in array)
			{
				if (!((Object)val == (Object)null))
				{
					((Behaviour)val).enabled = enabled;
					Text component = ((Component)val).GetComponent<Text>();
					if ((Object)component != (Object)null)
					{
						((Behaviour)component).enabled = enabled;
					}
				}
			}
			DmgTDisplay[] array2 = Resources.FindObjectsOfTypeAll<DmgTDisplay>();
			foreach (DmgTDisplay val2 in array2)
			{
				if (!((Object)val2 == (Object)null))
				{
					((Behaviour)val2).enabled = enabled;
					Text component2 = ((Component)val2).GetComponent<Text>();
					if ((Object)component2 != (Object)null)
					{
						((Behaviour)component2).enabled = enabled;
					}
				}
			}
			_statusLeft = (enabled ? "Damage debug overlay enabled." : "Damage debug overlay disabled.");
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Damage debug toggle failed: " + ex.GetBaseException().Message);
			_statusLeft = "Damage debug toggle failed.";
		}
	}

	private void SetModMicMute(bool enabled)
	{
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Expected O, but got Unknown
		//IL_0027: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		_modMicMuteActive = enabled;
		try
		{
			ModsMicMuteManager instance = ModsMicMuteManager.instance;
			IPlayer current = Player.Current;
			if ((Object)instance != (Object)null && current != null)
			{
				instance.SendRequest(new ModMuteArgument
				{
					modPlayerIdentifier = ((INetworkEntity)current).Identifier,
					isActive = enabled
				});
				_statusLeft = (enabled ? "Mod mic mute enabled." : "Mod mic mute disabled.");
			}
			else
			{
				_statusLeft = "Mod mic mute manager not ready.";
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Mod mic mute toggle failed: " + ex.GetBaseException().Message);
			_statusLeft = "Mod mic mute toggle failed.";
		}
	}

	private void UpdateLookRayHighlight()
	{
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Expected O, but got Unknown
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_006d: Expected O, but got Unknown
		//IL_0078: Unknown result type (might be due to invalid IL or missing references)
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Expected O, but got Unknown
		//IL_00c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d1: Expected O, but got Unknown
		//IL_00e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f3: Expected O, but got Unknown
		if (!_sceneReady)
		{
			ApplyLookHighlight(null);
			return;
		}
		Transform val = Actions.find_head();
		if ((Object)val == (Object)null)
		{
			PlayerController current = PlayerController.Current;
			Camera val2 = (((Object)(object)current != (Object)null) ? current.Camera : null) ?? Camera.main;
			val = (((Object)val2 != (Object)null) ? ((Component)val2).transform : null);
		}
		if ((Object)val == (Object)null)
		{
			ApplyLookHighlight(null);
			return;
		}
		int num = Physics.SphereCastNonAlloc(new Ray(val.position, val.forward), 0.12f, _lookRayHits, 1000f, -5, (QueryTriggerInteraction)1);
		if (num > 1)
		{
			SortLookRayHits(num);
		}
		Pickup pickup = null;
		for (int i = 0; i < num; i++)
		{
			Collider collider = ((RaycastHit)(ref _lookRayHits[i])).collider;
			if (!((Object)collider == (Object)null))
			{
				Pickup val3 = FindLookPickupFromHit(((Component)collider).transform, _lookIgnoreNameFragments);
				if ((Object)val3 != (Object)null)
				{
					pickup = val3;
					break;
				}
			}
		}
		ApplyLookHighlight(pickup);
	}

	private static void SortLookRayHits(int count)
	{
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_005a: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		for (int i = 1; i < count; i++)
		{
			RaycastHit val = _lookRayHits[i];
			float distance = ((RaycastHit)(ref val)).distance;
			int num = i - 1;
			while (num >= 0 && ((RaycastHit)(ref _lookRayHits[num])).distance > distance)
			{
				_lookRayHits[num + 1] = _lookRayHits[num];
				num--;
			}
			_lookRayHits[num + 1] = val;
		}
	}

	private static Pickup FindLookPickupFromHit(Transform start, string[] bannedFragments)
	{
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Expected O, but got Unknown
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Expected O, but got Unknown
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Expected O, but got Unknown
		Transform val = start;
		while ((Object)val != (Object)null)
		{
			if (LookNameContainsAny(val, bannedFragments))
			{
				return null;
			}
			if (((Object)val).name.IndexOf("Handle", StringComparison.OrdinalIgnoreCase) >= 0)
			{
				Pickup component = ((Component)val).GetComponent<Pickup>();
				if ((Object)component != (Object)null)
				{
					return component;
				}
			}
			Pickup component2 = ((Component)val).GetComponent<Pickup>();
			if ((Object)component2 != (Object)null)
			{
				return component2;
			}
			val = val.parent;
		}
		return null;
	}

	private static bool LookNameContainsAny(Transform t, string[] needles)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)t == (Object)null)
		{
			return false;
		}
		string name = ((Object)t).name;
		for (int i = 0; i < needles.Length; i++)
		{
			if (name.IndexOf(needles[i], StringComparison.OrdinalIgnoreCase) >= 0)
			{
				return true;
			}
		}
		return false;
	}

	private void ApplyLookHighlight(Pickup pickup)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Expected O, but got Unknown
		//IL_0016: Expected O, but got Unknown
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Expected O, but got Unknown
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_005d: Expected O, but got Unknown
		//IL_005d: Expected O, but got Unknown
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		//IL_006b: Expected O, but got Unknown
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Expected O, but got Unknown
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Expected O, but got Unknown
		//IL_0127: Unknown result type (might be due to invalid IL or missing references)
		//IL_0105: Unknown result type (might be due to invalid IL or missing references)
		//IL_010c: Expected O, but got Unknown
		//IL_01a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ae: Expected O, but got Unknown
		if ((Object)pickup == (Object)_lookCurrentPickup && (Object)pickup != (Object)null)
		{
			return;
		}
		bool flag = _lookOriginalMats.Count > 0 || _lookOriginalMPBs.Count > 0;
		if ((Object)pickup != (Object)_lookCurrentPickup || (Object)pickup == (Object)null)
		{
			if (flag)
			{
				RestoreLookHighlight();
			}
			_lookCurrentPickup = pickup;
		}
		if ((Object)pickup == (Object)null)
		{
			return;
		}
		Renderer[] componentsInChildren = ((Component)pickup).GetComponentsInChildren<Renderer>(false);
		if (componentsInChildren == null || componentsInChildren.Length == 0)
		{
			return;
		}
		EnsureLookFallbackMaterial();
		Renderer[] array = componentsInChildren;
		foreach (Renderer val in array)
		{
			if ((Object)val == (Object)null || val is ParticleSystemRenderer)
			{
				continue;
			}
			if (!_lookOriginalMats.ContainsKey(val))
			{
				_lookOriginalMats[val] = val.sharedMaterials.ToArray();
			}
			if (!_lookOriginalMPBs.ContainsKey(val))
			{
				MaterialPropertyBlock val2 = new MaterialPropertyBlock();
				val.GetPropertyBlock(val2);
				_lookOriginalMPBs[val] = val2;
			}
			if (!TryApplyLookEmission(val, LOOK_HIGHLIGHT_COLOR, 2.2f))
			{
				int num = Mathf.Max(1, val.sharedMaterials.Length);
				Material[] array2 = (Material[])(object)new Material[num];
				for (int j = 0; j < num; j++)
				{
					array2[j] = _lookFallbackHighlightMat;
				}
				val.materials = array2;
			}
			val.shadowCastingMode = (ShadowCastingMode)0;
			val.receiveShadows = false;
			Material[] materials = val.materials;
			foreach (Material val3 in materials)
			{
				if ((Object)val3 != (Object)null)
				{
					val3.enableInstancing = true;
				}
			}
		}
	}

	private void RestoreLookHighlight()
	{
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_004c: Expected O, but got Unknown
		if (_lookOriginalMats.Count == 0 && _lookOriginalMPBs.Count == 0)
		{
			_lookCurrentPickup = null;
			return;
		}
		foreach (KeyValuePair<Renderer, Material[]> lookOriginalMat in _lookOriginalMats)
		{
			Renderer key = lookOriginalMat.Key;
			if (!((Object)key == (Object)null))
			{
				key.sharedMaterials = lookOriginalMat.Value;
				if (_lookOriginalMPBs.TryGetValue(key, out var value))
				{
					key.SetPropertyBlock(value);
				}
				else
				{
					key.SetPropertyBlock((MaterialPropertyBlock)null);
				}
				key.shadowCastingMode = (ShadowCastingMode)1;
				key.receiveShadows = true;
			}
		}
		_lookOriginalMats.Clear();
		_lookOriginalMPBs.Clear();
		_lookCurrentPickup = null;
	}

	private bool TryApplyLookEmission(Renderer renderer, Color color, float intensity)
	{
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Expected O, but got Unknown
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Expected O, but got Unknown
		//IL_0108: Unknown result type (might be due to invalid IL or missing references)
		//IL_010a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Expected O, but got Unknown
		//IL_00f0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00db: Unknown result type (might be due to invalid IL or missing references)
		Material[] sharedMaterials = renderer.sharedMaterials;
		bool flag = false;
		bool flag2 = true;
		Material[] array = sharedMaterials;
		foreach (Material val in array)
		{
			if (!((Object)val == (Object)null))
			{
				if (val.HasProperty("_EmissionColor"))
				{
					flag = true;
				}
				if (!val.IsKeywordEnabled("_EMISSION"))
				{
					flag2 = false;
				}
			}
		}
		if (!flag || !flag2)
		{
			return false;
		}
		MaterialPropertyBlock val2 = new MaterialPropertyBlock();
		renderer.GetPropertyBlock(val2);
		string text = null;
		if (LookAnyMatHasProp(sharedMaterials, "_BaseColor"))
		{
			text = "_BaseColor";
		}
		else if (LookAnyMatHasProp(sharedMaterials, "_Color"))
		{
			text = "_Color";
		}
		if (!string.IsNullOrEmpty(text))
		{
			Color val3 = Color.white;
			array = sharedMaterials;
			foreach (Material val4 in array)
			{
				if ((Object)val4 != (Object)null && val4.HasProperty(text))
				{
					val3 = val4.GetColor(text);
					break;
				}
			}
			val2.SetColor(text, Color.Lerp(val3, color, 0.5f));
		}
		val2.SetColor("_EmissionColor", color * intensity);
		renderer.SetPropertyBlock(val2);
		return true;
	}

	private static bool LookAnyMatHasProp(Material[] mats, string prop)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Expected O, but got Unknown
		foreach (Material val in mats)
		{
			if ((Object)val != (Object)null && val.HasProperty(prop))
			{
				return true;
			}
		}
		return false;
	}

	private void EnsureLookFallbackMaterial()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Expected O, but got Unknown
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Expected O, but got Unknown
		if (!((Object)_lookFallbackHighlightMat != (Object)null))
		{
			Shader val = Shader.Find("Unlit/Color");
			if ((Object)val == (Object)null)
			{
				val = Shader.Find("Sprites/Default");
			}
			_lookFallbackHighlightMat = new Material(val)
			{
				color = LOOK_HIGHLIGHT_COLOR,
				renderQueue = 3000
			};
			if (_lookFallbackHighlightMat.HasProperty("_ZWrite"))
			{
				_lookFallbackHighlightMat.SetInt("_ZWrite", 0);
			}
		}
	}

	private void UpdatePawgleDocks()
	{
		if (!_sceneReady)
		{
			return;
		}
		Pickup[] array = Object.FindObjectsOfType<Pickup>(true);
		if (array != null && array.Length != 0)
		{
			EnableDockedPickupsAlways(array);
			if (!(Time.time < _pawgleNextScanTime))
			{
				_pawgleNextScanTime = Time.time + 1f;
				EnableLoosePickups(array);
			}
		}
	}

	private static void EnableDockedPickupsAlways(Pickup[] pickups)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Expected O, but got Unknown
		foreach (Pickup val in pickups)
		{
			if (!((Object)val == (Object)null) && PawgleIsInDock(((Component)val).transform))
			{
				((Behaviour)val).enabled = true;
				Collider[] componentsInChildren = ((Component)val).GetComponentsInChildren<Collider>(true);
				for (int j = 0; j < componentsInChildren.Length; j++)
				{
					componentsInChildren[j].enabled = true;
				}
			}
		}
	}

	private static void EnableLoosePickups(Pickup[] pickups)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Expected O, but got Unknown
		foreach (Pickup val in pickups)
		{
			if ((Object)val == (Object)null)
			{
				continue;
			}
			Transform transform = ((Component)val).transform;
			if (PawgleIsInDock(transform) || PawgleIsInIgnoredContainer(transform))
			{
				continue;
			}
			if (!((Behaviour)val).enabled)
			{
				((Behaviour)val).enabled = true;
			}
			Collider[] componentsInChildren = ((Component)val).GetComponentsInChildren<Collider>(true);
			for (int j = 0; j < componentsInChildren.Length; j++)
			{
				if (!componentsInChildren[j].enabled)
				{
					componentsInChildren[j].enabled = true;
				}
			}
		}
	}

	private static bool PawgleIsInIgnoredContainer(Transform obj)
	{
		//IL_003c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Expected O, but got Unknown
		Transform val = obj;
		while ((Object)val != (Object)null)
		{
			string name = ((Object)val).name;
			if (name.Contains("Pouch(Clone)") || name.Contains("Quiver") || name.Contains("Lantern"))
			{
				return true;
			}
			val = val.parent;
		}
		return false;
	}

	private static bool PawgleIsInDock(Transform obj)
	{
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Expected O, but got Unknown
		Transform val = obj;
		while ((Object)val != (Object)null)
		{
			if (((Object)val).name.Contains("Dock"))
			{
				return true;
			}
			val = val.parent;
		}
		return false;
	}

	public override void OnLateUpdate()
	{
		PlayerController current = PlayerController.Current;
		Camera val = (((Object)(object)current != (Object)null) ? current.Camera : null) ?? Camera.main;
		Transform viewTransform = (((Object)(object)val != (Object)null) ? ((Component)val).transform : null);
		if (CSRuntime.IsAlive)
		{
			CSRuntime.SetViewTransform(viewTransform);
		}
	}

	public override void OnGUI()
	{
		DrawSimpleUi();
	}

	private void DrawSimpleUi()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Expected O, but got Unknown
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Unknown result type (might be due to invalid IL or missing references)
		_simpleUiRect = GUILayout.Window(9137, _simpleUiRect, (WindowFunction)delegate
		{
			//IL_00fd: Unknown result type (might be due to invalid IL or missing references)
			//IL_0117: Unknown result type (might be due to invalid IL or missing references)
			//IL_0179: Unknown result type (might be due to invalid IL or missing references)
			GUILayout.Label("Join Server", Array.Empty<GUILayoutOption>());
			GUILayout.Space(4f);
			GUILayout.Label("Allowed Servers", Array.Empty<GUILayoutOption>());
			if (AllowedServers.Count == 0)
			{
				GUILayout.Label("No allowed servers configured.", Array.Empty<GUILayoutOption>());
			}
			else
			{
				foreach (KeyValuePair<int, string> allowedServer in AllowedServers)
				{
					if (GUILayout.Button($"{allowedServer.Value} ({allowedServer.Key})", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
					{
						if (!IsServerAllowed(allowedServer.Key))
						{
							QuitForDisallowed("Blocked UI join attempt: " + allowedServer.Key);
						}
						else
						{
							JoinAllowedServer(allowedServer.Key);
						}
					}
				}
			}
			if (!string.IsNullOrEmpty(_connectErr))
			{
				GUI.contentColor = Color.red;
				GUILayout.Label(_connectErr, Array.Empty<GUILayoutOption>());
				GUI.contentColor = Color.white;
			}
			GUILayout.Space(6f);
			GUILayout.Label("Teleports", Array.Empty<GUILayoutOption>());
			foreach (KeyValuePair<string, Vector3> teleport in _teleports)
			{
				if (GUILayout.Button(teleport.Key, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
				{
					TeleportLocalPlayer(teleport.Value, (LocomotionFunction)0);
				}
			}
			GUILayout.Space(6f);
			if (GUILayout.Button("Quit Game", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(26f) }))
			{

			}
			GUI.DragWindow();
		}, "Base Model", Array.Empty<GUILayoutOption>());
	}

private static bool IsServerAllowed(int serverId)
{
    return true;
}

private async void JoinAllowedServer(int serverId)
{
    // Bypass any disallowed check
    try
    {
        GameModeManager.JoinServer(await ApiAccess.ApiClient.ServerClient.GetServerAsync(serverId));
    }
    catch (Exception ex)
    {
        MelonLogger.Warning("Join error: " + ex.GetBaseException().Message);
    }
}
	private void DrawVoiceSliders()
	{
		//IL_0077: Unknown result type (might be due to invalid IL or missing references)
		//IL_0094: Unknown result type (might be due to invalid IL or missing references)
		//IL_0099: Unknown result type (might be due to invalid IL or missing references)
		GUILayout.Label("Voice (Vivox):", Array.Empty<GUILayoutOption>());
		GUILayout.Label("Local volume (-10 to 50 dB)", Array.Empty<GUILayoutOption>());
		List<IPlayer> list = Player.AllPlayers?.OrderBy((IPlayer p) => p.UserInfo.Identifier).ToList() ?? new List<IPlayer>();
		if (list.Count == 0)
		{
			GUILayout.Label("No players online.", Array.Empty<GUILayoutOption>());
			return;
		}
		_voiceScroll = GUILayout.BeginScrollView(_voiceScroll, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(200f) });
		foreach (IPlayer item in list)
		{
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.Label($"{item.UserInfo.Username} ({item.UserInfo.Identifier})", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.ExpandWidth(true) });
			int num = 0;
			try
			{
				IParticipant voice = item.Voice;
				num = ((voice != null) ? ((IParticipantProperties)voice).LocalVolumeAdjustment : 0);
			}
			catch
			{
			}
			GUILayout.Label($"{num} dB", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
			int num2 = Mathf.RoundToInt(GUILayout.HorizontalSlider((float)num, -10f, 50f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(200f) }));
			if (num2 != num)
			{
				try
				{
					if (((item != null) ? item.Voice : null) != null)
					{
						((IParticipantProperties)item.Voice).LocalVolumeAdjustment = num2;
					}
				}
				catch (Exception ex)
				{
					MelonLogger.Warning("[Voice] Failed to set volume: " + ex.GetBaseException().Message);
				}
			}
			GUILayout.EndHorizontal();
		}
		GUILayout.EndScrollView();
	}

	private static void QuitForDisallowed(string reason)
	{
		MelonLogger.Error("Disallowed server join attempt. " + reason);
	}

	private void InitStyles()
	{
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Expected O, but got Unknown
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		//IL_0109: Expected O, but got Unknown
		//IL_0123: Unknown result type (might be due to invalid IL or missing references)
		//IL_013f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0144: Unknown result type (might be due to invalid IL or missing references)
		//IL_014c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0153: Unknown result type (might be due to invalid IL or missing references)
		//IL_0158: Unknown result type (might be due to invalid IL or missing references)
		//IL_0162: Expected O, but got Unknown
		//IL_0167: Expected O, but got Unknown
		//IL_016e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0173: Unknown result type (might be due to invalid IL or missing references)
		//IL_017b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0182: Unknown result type (might be due to invalid IL or missing references)
		//IL_0189: Unknown result type (might be due to invalid IL or missing references)
		//IL_018e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0198: Expected O, but got Unknown
		//IL_019d: Expected O, but got Unknown
		//IL_01a4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a9: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ce: Expected O, but got Unknown
		//IL_01d3: Expected O, but got Unknown
		//IL_01de: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01fa: Expected O, but got Unknown
		//IL_01fa: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ff: Unknown result type (might be due to invalid IL or missing references)
		//IL_0209: Expected O, but got Unknown
		//IL_020e: Expected O, but got Unknown
		//IL_0219: Unknown result type (might be due to invalid IL or missing references)
		//IL_021e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0226: Unknown result type (might be due to invalid IL or missing references)
		//IL_022b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0235: Expected O, but got Unknown
		//IL_0235: Unknown result type (might be due to invalid IL or missing references)
		//IL_023a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0244: Expected O, but got Unknown
		//IL_0249: Expected O, but got Unknown
		//IL_0250: Unknown result type (might be due to invalid IL or missing references)
		//IL_0255: Unknown result type (might be due to invalid IL or missing references)
		//IL_025c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0267: Unknown result type (might be due to invalid IL or missing references)
		//IL_026c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0276: Expected O, but got Unknown
		//IL_027b: Expected O, but got Unknown
		//IL_0285: Unknown result type (might be due to invalid IL or missing references)
		//IL_028b: Expected O, but got Unknown
		//IL_02a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ab: Expected O, but got Unknown
		//IL_02b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ba: Expected O, but got Unknown
		//IL_02bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c9: Expected O, but got Unknown
		if (_btn == null || _h2 == null || _h3 == null)
		{
			Color c = default(Color);
			((Color)(ref c))._002Ector(0.16f, 0.16f, 0.18f, 0.98f);
			Color c2 = default(Color);
			((Color)(ref c2))._002Ector(1f, 1f, 1f, 0.06f);
			_texPanel = MakeTex(2, 2, c);
			_texDivider = MakeTex(2, 2, c2);
			_pfv3RectFill = MakeTex(2, 2, new Color(0.2f, 0.6f, 1f, 0.1f));
			_pfv3RectBorder = MakeTex(2, 2, new Color(1f, 1f, 1f, 0.25f));
			_label = new GUIStyle(GUI.skin.label)
			{
				fontSize = 12,
				wordWrap = true
			};
			GUIStyle val = new GUIStyle(GUI.skin.label)
			{
				fontSize = 10
			};
			val.normal.textColor = new Color(0.78f, 0.78f, 0.78f, 1f);
			_small = val;
			_h1 = new GUIStyle(GUI.skin.label)
			{
				fontSize = 14,
				fontStyle = (FontStyle)1,
				padding = new RectOffset(1, 1, 1, 1)
			};
			_h2 = new GUIStyle(_h1)
			{
				fontSize = 13,
				fontStyle = (FontStyle)1,
				alignment = (TextAnchor)3,
				padding = new RectOffset(2, 2, 2, 0)
			};
			_h3 = new GUIStyle(_small)
			{
				fontSize = 11,
				fontStyle = (FontStyle)1,
				alignment = (TextAnchor)3,
				padding = new RectOffset(2, 2, 1, 1)
			};
			_btn = new GUIStyle(GUI.skin.button)
			{
				fontSize = 12,
				padding = new RectOffset(8, 8, 5, 5),
				margin = new RectOffset(3, 3, 3, 3)
			};
			_topTab = new GUIStyle(GUI.skin.button)
			{
				fontSize = 12,
				padding = new RectOffset(8, 8, 5, 5),
				margin = new RectOffset(1, 1, 0, 0)
			};
			_sidebarBtn = new GUIStyle(_btn)
			{
				alignment = (TextAnchor)3,
				fixedHeight = 28f,
				padding = new RectOffset(8, 8, 4, 4)
			};
			GUIStyle val2 = new GUIStyle(GUI.skin.box);
			val2.normal.background = _texPanel;
			val2.padding = new RectOffset(6, 6, 5, 5);
			val2.margin = new RectOffset(0, 0, 4, 4);
			val2.border = new RectOffset(6, 6, 6, 6);
			_card = val2;
		}
	}

	private void RefreshJoinableServerIds()
	{
		_joinableServerIds.Clear();
		foreach (int defaultJoinableServerId in _defaultJoinableServerIds)
		{
			_joinableServerIds.Add(defaultJoinableServerId);
		}
		foreach (ServerEntry item in _saved)
		{
			if (int.TryParse(item.Id, NumberStyles.Integer, CultureInfo.InvariantCulture, out var result) && result > 0)
			{
				_joinableServerIds.Add(result);
			}
		}
	}

private static bool IsJoinableServerId(int serverId, out string message)
{
    message = "";
    return true;
}
	private static Task<ServerJoinResult> BuildDeniedJoinResult(string message, int serverId)
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Expected O, but got Unknown
		ServerJoinResult result = null;
		try
		{
			result = (ServerJoinResult)FormatterServices.GetUninitializedObject(typeof(ServerJoinResult));
			SetServerJoinResultValue(result, "IsAllowed", false);
			SetServerJoinResultValue(result, "Message", message);
			SetServerJoinResultValue(result, "ServerIdentifier", serverId);
			SetServerJoinResultValue(result, "FailReason", "NotAllowed");
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] Failed to build denied join result: " + ex.GetBaseException().Message);
		}
		return Task.FromResult<ServerJoinResult>(result);
	}

	private static void SetServerJoinResultValue(ServerJoinResult result, string name, object value)
	{
		if (result == null || string.IsNullOrWhiteSpace(name))
		{
			return;
		}
		Type typeFromHandle = typeof(ServerJoinResult);
		PropertyInfo property = typeFromHandle.GetProperty(name, BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (property != null && property.CanWrite)
		{
			object value2 = ConvertJoinResultValue(property.PropertyType, value);
			property.SetValue(result, value2);
			return;
		}
		FieldInfo field = typeFromHandle.GetField(name, BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (field != null)
		{
			object value3 = ConvertJoinResultValue(field.FieldType, value);
			field.SetValue(result, value3);
		}
	}

	private static object ConvertJoinResultValue(Type targetType, object value)
	{
		if (targetType == null)
		{
			return value;
		}
		if (value == null)
		{
			return null;
		}
		Type underlyingType = Nullable.GetUnderlyingType(targetType);
		if (underlyingType != null)
		{
			targetType = underlyingType;
		}
		if (targetType.IsEnum)
		{
			if (value is string value2)
			{
				try
				{
					return Enum.Parse(targetType, value2, ignoreCase: true);
				}
				catch
				{
					Array values = Enum.GetValues(targetType);
					return (values.Length > 0) ? values.GetValue(0) : null;
				}
			}
			return Enum.ToObject(targetType, value);
		}
		if (targetType.IsInstanceOfType(value))
		{
			return value;
		}
		return Convert.ChangeType(value, targetType, CultureInfo.InvariantCulture);
	}

	private async void JoinServerSimple()
	{
		_connectErr = "";
		if (!int.TryParse(_srvId, out var result) || result <= 0)
		{
			_connectErr = "Invalid Server ID";
			return;
		}
		if (!IsJoinableServerId(result, out var message))
		{
			_connectErr = message;
			return;
		}
		try
		{
			GameServerInfo val = await ApiAccess.ApiClient.ServerClient.GetServerAsync(result);
			Basemodal.main_values.pc2q = true;
			if (val.Target == 1)
			{
				Basemodal.main_values.pc2q = false;
			}
			Basemodal.main_values.is_connected = true;
			Basemodal.main_values.target_server_id = val.Identifier;
			Basemodal.main_values.is_keyboard_movement = false;
			PlayerController current = PlayerController.Current;
			if ((Object)(object)current != (Object)null)
			{
				PlayerMessageDisplay messageDisplay = current.MessageDisplay;
				if ((Object)(object)messageDisplay != (Object)null)
				{
					messageDisplay.Display($"connecting to server {val.Name}({val.Identifier})", 7f, (DisplayMessageType)2);
				}
			}
			GameModeManager.JoinServer(val);
		}
		catch (Exception ex)
		{
			_connectErr = "Join error: " + ex.GetBaseException().Message;
		}
	}

	private void LoadPersisted()
	{
		try
		{
			if (!File.Exists(_savePath))
			{
				_save = new SaveBlob();
				RefreshJoinableServerIds();
				return;
			}
			string text = File.ReadAllText(_savePath);
			SaveBlob saveBlob = null;
			try
			{
				saveBlob = JsonConvert.DeserializeObject<SaveBlob>(text);
			}
			catch
			{
				saveBlob = JsonUtility.FromJson<SaveBlob>(text);
			}
			_save = saveBlob ?? new SaveBlob();
			if (_save.Servers == null)
			{
				RefreshJoinableServerIds();
				return;
			}
			foreach (SavedServer server in _save.Servers)
			{
				if (!string.IsNullOrWhiteSpace(server?.Id))
				{
					_saved.Add(new ServerEntry
					{
						Name = (server.Name ?? ""),
						Id = server.Id
					});
				}
			}
			RefreshJoinableServerIds();
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] LoadPersisted failed: " + ex.GetBaseException().Message);
			_save = new SaveBlob();
			RefreshJoinableServerIds();
		}
	}

	private void SavePersisted()
	{
		try
		{
			_save.Servers = (from e in _saved
				where !string.IsNullOrWhiteSpace(e.Id)
				select new SavedServer
				{
					Name = (e.Name ?? ""),
					Id = e.Id
				}).ToList();
			RefreshJoinableServerIds();
			string directoryName = Path.GetDirectoryName(_savePath);
			if (!string.IsNullOrEmpty(directoryName))
			{
				Directory.CreateDirectory(directoryName);
			}
			string contents = JsonConvert.SerializeObject((object)_save, (Formatting)1);
			File.WriteAllText(_savePath, contents);
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] SavePersisted failed: " + ex.GetBaseException().Message);
		}
	}

	private async void InitializeApiFromSavedToken()
	{
		if (_api == null)
		{
			_api = new BasemodalApiInteractor();
		}
		if (!string.IsNullOrWhiteSpace(_save.ApiToken))
		{
			_awaitingToken = true;
			_tokenInput = _save.ApiToken;
			await SetupApiTokenAsync(_save.ApiToken);
			return;
		}
		_awaitingToken = true;
		_tokenInput = "";
		AddChatMessage("system", "No API token set. Press T, paste your token into chat and press Enter - it will be saved.", Color.yellow);
		_chatOverlayVisible = true;
		_chatTyping = true;
		_chatFocusPending = true;
		_tokenFocusPending = true;
	}

	private bool NeedToken()
	{
		if (_api != null)
		{
			return !_api.IsAuthenticated;
		}
		return true;
	}

	private async Task SetupApiTokenAsync(string token)
	{
		try
		{
			_tokenSubmitting = true;
			string raw = ExtractRawToken(token);
			if (string.IsNullOrWhiteSpace(raw))
			{
				_awaitingToken = true;
				_tokenSubmitting = false;
				AddChatMessage("system", "Token is empty. Try again.", Color.red);
				return;
			}
			IHighLevelApiClient apiClient = ApiAccess.ApiClient;
			object obj;
			if (apiClient == null)
			{
				obj = null;
			}
			else
			{
				IUserApiClient userClient = apiClient.UserClient;
				obj = ((userClient != null) ? userClient.LoggedInUserInfo : null);
			}
			object obj2 = obj;
			string username = ((obj2 != null) ? ((UserInfo)obj2).Username : null) ?? "me";
			int identifier = ((obj != null) ? ((UserInfo)obj).Identifier : 0);
			if (_api == null)
			{
				_api = new BasemodalApiInteractor();
			}
			HookApiEvents();
			await _api.ConnectAndLoginAsync(raw, username, identifier);
			_save.ApiToken = raw;
			try
			{
				SavePersisted();
			}
			catch
			{
			}
			_awaitingToken = false;
			_tokenInput = raw;
			_tokenSubmitting = false;
			MelonLogger.Msg("[Auth] Connected via API and authenticated.");
		}
		catch (Exception ex)
		{
			_awaitingToken = true;
			_tokenSubmitting = false;
			AddChatMessage("system", "Token login failed: " + ex.GetBaseException().Message, Color.red);
			MelonLogger.Warning("[Auth] API login failed: " + ex.GetBaseException().Message);
		}
	}

	private void OnGrabPressed(bool right)
	{
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Expected O, but got Unknown
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Expected O, but got Unknown
		//IL_0080: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Expected O, but got Unknown
		//IL_0087: Unknown result type (might be due to invalid IL or missing references)
		//IL_0092: Expected O, but got Unknown
		if (snapFollowPrimed)
		{
			SelectFollowTarget(right);
		}
		else
		{
			if (TryGrabLookTarget(right))
			{
				return;
			}
			PlayerController current = PlayerController.Current;
			PlayerCharacter val = (PlayerCharacter)((current is PlayerCharacter) ? current : null);
			if (!((Object)val == (Object)null))
			{
				object obj;
				if (!right)
				{
					Controller leftController = ((PlayerController)val).LeftController;
					obj = (((Object)(object)leftController != (Object)null) ? leftController.Interactor : null);
				}
				else
				{
					Controller rightController = ((PlayerController)val).RightController;
					obj = (((Object)(object)rightController != (Object)null) ? rightController.Interactor : null);
				}
				Interactor val2 = (Interactor)obj;
				if (!((Object)val2 == (Object)null))
				{
					TrySnatchFromNearbyHands(val2, !right);
				}
			}
		}
	}

	private bool TryGrabLookTarget(bool rightHand)
	{
		//IL_0008: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Expected O, but got Unknown
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ae: Expected O, but got Unknown
		//IL_00ae: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b9: Expected O, but got Unknown
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_002f: Expected O, but got Unknown
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Expected O, but got Unknown
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0072: Expected O, but got Unknown
		Pickup lookCurrentPickup = _lookCurrentPickup;
		if ((Object)lookCurrentPickup == (Object)null)
		{
			return false;
		}
		PlayerController current = PlayerController.Current;
		PlayerCharacter val = (PlayerCharacter)((current is PlayerCharacter) ? current : null);
		if ((Object)val == (Object)null)
		{
			return false;
		}
		Controller val2 = (rightHand ? ((PlayerController)val).RightController : ((PlayerController)val).LeftController);
		Interactor val3 = (((Object)(object)val2 != (Object)null) ? val2.Interactor : null);
		if ((Object)val3 == (Object)null || val3.IsInteracting)
		{
			return false;
		}
		try
		{
			if (lookCurrentPickup.IsDocked)
			{
				lookCurrentPickup.Undock(val3, true, 1);
			}
		}
		catch
		{
		}
		val3.ResetTimeout();
		if ((Object)val3.StartInteract((Interactable)lookCurrentPickup, false, false, false) != (Object)null)
		{
			RestoreLookHighlight();
			return true;
		}
		return false;
	}

	public static void ToggleStorage()
	{
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Expected O, but got Unknown
		GameObject[] array = Object.FindObjectsOfType<GameObject>();
		foreach (GameObject val in array)
		{
			if (((Object)val).name.EndsWith("Bag(Clone)"))
			{
				Transform val2 = val.transform.Find("Storage");
				if ((Object)val2 != (Object)null)
				{
					bool activeSelf = ((Component)val2).gameObject.activeSelf;
					((Component)val2).gameObject.SetActive(!activeSelf);
				}
			}
		}
	}

	private void TryTeleportToCustomCoords()
	{
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		if (TryParseVector3(_customTeleportInput, out var result))
		{
			TeleportLocalPlayer(result, (LocomotionFunction)0);
			MelonLogger.Msg($"[Teleport] Teleported to custom coordinates: {result}");
		}
		else
		{
			MelonLogger.Warning("[Teleport] Invalid coordinate input: '" + _customTeleportInput + "'. Please use format 'X,Y,Z'.");
		}
	}

	private bool TryParseVector3(string input, out Vector3 result)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		result = Vector3.zero;
		string[] array = input.Split(new char[3] { ',', ' ', '\t' }, StringSplitOptions.RemoveEmptyEntries);
		if (array.Length != 3)
		{
			return false;
		}
		if (float.TryParse(array[0], out var result2) && float.TryParse(array[1], out var result3) && float.TryParse(array[2], out var result4))
		{
			result = new Vector3(result2, result3, result4);
			return true;
		}
		return false;
	}

	private bool TrySnatchFromNearbyHands(Interactor myInter, bool preferLeftOnSelf)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_0034: Expected O, but got Unknown
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0043: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Expected O, but got Unknown
		//IL_01a2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ad: Expected O, but got Unknown
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0087: Unknown result type (might be due to invalid IL or missing references)
		//IL_0091: Expected O, but got Unknown
		//IL_0091: Expected O, but got Unknown
		//IL_00cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d6: Expected O, but got Unknown
		//IL_010f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0116: Expected O, but got Unknown
		//IL_0118: Unknown result type (might be due to invalid IL or missing references)
		//IL_0123: Expected O, but got Unknown
		//IL_0155: Unknown result type (might be due to invalid IL or missing references)
		//IL_0160: Expected O, but got Unknown
		//IL_0127: Unknown result type (might be due to invalid IL or missing references)
		//IL_012c: Unknown result type (might be due to invalid IL or missing references)
		//IL_012d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0132: Unknown result type (might be due to invalid IL or missing references)
		//IL_0164: Unknown result type (might be due to invalid IL or missing references)
		//IL_0169: Unknown result type (might be due to invalid IL or missing references)
		//IL_016a: Unknown result type (might be due to invalid IL or missing references)
		//IL_016f: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)myInter == (Object)null)
		{
			return false;
		}
		PlayerController current = PlayerController.Current;
		PlayerCharacter val = (PlayerCharacter)((current is PlayerCharacter) ? current : null);
		if ((Object)val == (Object)null)
		{
			return false;
		}
		Vector3 position = ((Component)myInter).transform.position;
		PlayerCharacter val2 = null;
		HandSelector whichHand = HandSelector.Any;
		float num = float.MaxValue;
		float num2 = 0.20249999f;
		PlayerCharacter[] array = Object.FindObjectsOfType<PlayerCharacter>();
		foreach (PlayerCharacter val3 in array)
		{
			if ((Object)val3 == (Object)null || (Object)val3 == (Object)val)
			{
				continue;
			}
			Controller leftController = ((PlayerController)val3).LeftController;
			object obj;
			if ((Object)(object)leftController == (Object)null)
			{
				obj = null;
			}
			else
			{
				Hand hand = leftController.Hand;
				obj = (((Object)(object)hand != (Object)null) ? ((Component)hand).transform : null);
			}
			Transform val4 = (Transform)obj;
			Controller rightController = ((PlayerController)val3).RightController;
			object obj2;
			if ((Object)(object)rightController == (Object)null)
			{
				obj2 = null;
			}
			else
			{
				Hand hand2 = rightController.Hand;
				obj2 = (((Object)(object)hand2 != (Object)null) ? ((Component)hand2).transform : null);
			}
			Transform val5 = (Transform)obj2;
			Vector3 val6;
			if ((Object)val4 != (Object)null)
			{
				val6 = val4.position - position;
				float sqrMagnitude = ((Vector3)(ref val6)).sqrMagnitude;
				if (sqrMagnitude <= num2 && sqrMagnitude < num)
				{
					num = sqrMagnitude;
					val2 = val3;
					whichHand = HandSelector.Left;
				}
			}
			if ((Object)val5 != (Object)null)
			{
				val6 = val5.position - position;
				float sqrMagnitude2 = ((Vector3)(ref val6)).sqrMagnitude;
				if (sqrMagnitude2 <= num2 && sqrMagnitude2 < num)
				{
					num = sqrMagnitude2;
					val2 = val3;
					whichHand = HandSelector.Right;
				}
			}
		}
		if ((Object)val2 == (Object)null)
		{
			return false;
		}
		return GrabFromHandStore(val, val2, whichHand, preferLeftOnSelf);
	}

	private bool GrabFromHandStore(PlayerCharacter self, PlayerCharacter target, HandSelector whichHand, bool preferLeftOnSelf)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Expected O, but got Unknown
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Expected O, but got Unknown
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Expected O, but got Unknown
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a2: Expected O, but got Unknown
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c3: Expected O, but got Unknown
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Expected O, but got Unknown
		if ((Object)self == (Object)null || (Object)target == (Object)null)
		{
			return false;
		}
		HandStore val = new HandStore(((PlayerController)target).LeftController);
		HandStore val2 = new HandStore(((PlayerController)target).RightController);
		HandStore[] array = whichHand switch
		{
			HandSelector.Left => (HandStore[])(object)new HandStore[1] { val }, 
			HandSelector.Right => (HandStore[])(object)new HandStore[1] { val2 }, 
			_ => (HandStore[])(object)new HandStore[2] { val, val2 }, 
		};
		for (int i = 0; i < array.Length; i++)
		{
			if (TryPopFrom(array[i], out var pickup))
			{
				Interactor freeInteractor = GetFreeInteractor(self, preferLeftOnSelf);
				if ((Object)freeInteractor == (Object)null)
				{
					SafeDropNearSelf(self, pickup);
					return false;
				}
				if ((Object)freeInteractor.StartInteract((Interactable)pickup, NetworkSceneManager.IsServer, true, false) != (Object)null)
				{
					return true;
				}
				SafeDropNearSelf(self, pickup);
				return false;
			}
		}
		return false;
	}

	private static bool TryPopFrom(HandStore hs, out Pickup pickup)
	{
		if (hs.TryPop(ref pickup))
		{
			return true;
		}
		Pickup val = null;
		if (hs.TryPeek(ref val) && hs.TryPop(ref pickup))
		{
			return true;
		}
		pickup = null;
		return false;
	}

	private static Interactor GetFreeInteractor(PlayerCharacter self, bool preferLeft)
	{
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0046: Expected O, but got Unknown
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Expected O, but got Unknown
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0098: Expected O, but got Unknown
		//IL_00a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Expected O, but got Unknown
		object obj;
		if (!preferLeft)
		{
			Controller rightController = ((PlayerController)self).RightController;
			obj = (((Object)(object)rightController != (Object)null) ? rightController.Interactor : null);
		}
		else
		{
			Controller leftController = ((PlayerController)self).LeftController;
			obj = (((Object)(object)leftController != (Object)null) ? leftController.Interactor : null);
		}
		Interactor val = (Interactor)obj;
		object obj2;
		if (!preferLeft)
		{
			Controller leftController2 = ((PlayerController)self).LeftController;
			obj2 = (((Object)(object)leftController2 != (Object)null) ? leftController2.Interactor : null);
		}
		else
		{
			Controller rightController2 = ((PlayerController)self).RightController;
			obj2 = (((Object)(object)rightController2 != (Object)null) ? rightController2.Interactor : null);
		}
		Interactor val2 = (Interactor)obj2;
		if ((Object)val != (Object)null && !val.IsInteracting)
		{
			return val;
		}
		if ((Object)val2 != (Object)null && !val2.IsInteracting)
		{
			return val2;
		}
		return null;
	}

	private static void SafeDropNearSelf(PlayerCharacter self, Pickup p)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Expected O, but got Unknown
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Expected O, but got Unknown
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		if (!((Object)p == (Object)null) && !((Object)self == (Object)null))
		{
			Vector3 position = ((Component)self).transform.position + Vector3.up * 0.2f;
			Rigidbody component = ((Component)p).GetComponent<Rigidbody>();
			if ((Object)component != (Object)null)
			{
				component.isKinematic = false;
				component.velocity = Vector3.zero;
				component.angularVelocity = Vector3.zero;
			}
			((Component)p).transform.position = position;
		}
	}

	private string GetTpLabel()
	{
		if (_tpOptions == null)
		{
			_tpOptions = _teleports.Keys.ToArray();
		}
		if (_tpSelected >= 0 && _tpSelected < _tpOptions.Length)
		{
			return _tpOptions[_tpSelected];
		}
		return "Select Destination";
	}

	private void ProcessAutoUndock()
	{
		//IL_0005: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Expected O, but got Unknown
		if (!((Object)PlayerController.Current == (Object)null))
		{
			OpenXRControllerInput val = Actions.find_controller(right: false);
			OpenXRControllerInput val2 = Actions.find_controller(right: true);
			bool flag = val != null && ((InputSource)((ControllerInput)val).Grab).State;
			bool num = val2 != null && ((InputSource)((ControllerInput)val2).Grab).State;
			Mouse current = Mouse.current;
			bool flag2 = current != null && current.leftButton.isPressed;
			bool flag3 = current != null && current.rightButton.isPressed;
			if (PointerOverUnifiedUI())
			{
				flag2 = false;
				flag3 = false;
			}
			bool flag4 = flag || flag2;
			bool flag5 = num || flag3;
			if (flag4 && !_prevLeftGrab)
			{
				OnGrabPressed(right: false);
			}
			if (flag5 && !_prevRightGrab)
			{
				OnGrabPressed(right: true);
			}
			_prevLeftGrab = flag4;
			_prevRightGrab = flag5;
		}
	}

	private void BHSetEnabled(bool on)
	{
		if (_bhEnabled != on)
		{
			_bhEnabled = on;
		}
	}

	private void BHToggle()
	{
		BHSetEnabled(!_bhEnabled);
	}

	private void EnsureWheel()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Expected O, but got Unknown
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Expected O, but got Unknown
		if (!((Object)_wheelGO != (Object)null) || !((Object)_wheel != (Object)null))
		{
			_wheelGO = new GameObject("Unified_RadialWheel");
			_wheel = _wheelGO.AddComponent<RadialMenu3D>();
			_wheel.Build(0.045f, 0.135f, 12);
			_wheelGO.layer = LayerMask.NameToLayer("UI");
			_wheelGO.transform.localScale = Vector3.one * 1f;
			BuildWheelPage(WheelPage.Root);
		}
	}

	private void ShowWheel(bool show)
	{
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Expected O, but got Unknown
		//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f7: Expected O, but got Unknown
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Expected O, but got Unknown
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Expected O, but got Unknown
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		EnsureWheel();
		_wheelVisible = show;
		if ((Object)_wheelGO != (Object)null)
		{
			_wheelGO.SetActive(show);
		}
		_wheelHoverIndex = -1;
		_wheelHoverStart = -1f;
		if (show)
		{
			BuildWheelPage(_wheelPage = WheelPage.Root);
			Transform val = Actions.find_hand(right: true);
			if ((Object)val != (Object)null && (Object)_wheelGO != (Object)null)
			{
				_wheelGO.transform.SetParent(val, false);
				_wheelGO.transform.localPosition = new Vector3(0.06f, 0.02f, 0.1f);
				_wheelGO.transform.localRotation = Quaternion.identity;
				int layer = ((Component)val).gameObject.layer;
				SetLayerRecursively(_wheelGO, layer);
			}
			PositionWheelAtRightHand();
		}
		else if ((Object)_wheelGO != (Object)null)
		{
			_wheelGO.transform.SetParent((Transform)null, true);
		}
	}

	private void PositionWheelAtRightHand()
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Expected O, but got Unknown
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0056: Expected O, but got Unknown
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Expected O, but got Unknown
		//IL_0084: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Unknown result type (might be due to invalid IL or missing references)
		//IL_0099: Unknown result type (might be due to invalid IL or missing references)
		//IL_009e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		if (_wheelVisible && !((Object)_wheelGO == (Object)null))
		{
			Transform val = Actions.find_hand(right: true);
			PlayerController current = PlayerController.Current;
			Camera val2 = (((Object)(object)current != (Object)null) ? current.Camera : null) ?? Camera.main;
			if (!((Object)val == (Object)null) && !((Object)val2 == (Object)null))
			{
				_wheelGO.transform.position = val.TransformPoint(new Vector3(0.06f, 0.02f, 0.1f));
				Vector3 forward = ((Component)val2).transform.forward;
				Vector3 up = ((Component)val2).transform.up;
				_wheelGO.transform.rotation = Quaternion.LookRotation(forward, up);
				_wheelGO.transform.localScale = Vector3.one * 0.5f;
			}
		}
	}

	private void UpdateWheelSelection()
	{
		//IL_000e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Expected O, but got Unknown
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		if (!_wheelVisible || (Object)_wheel == (Object)null)
		{
			return;
		}
		Vector2 axis = _xrRStickAxis;
		if (((Vector2)(ref axis)).sqrMagnitude < 0.0001f && Gamepad.current != null && InputControlExtensions.IsActuated((InputControl)Gamepad.current.rightStick, 0f))
		{
			axis = ((InputControl<Vector2>)(object)Gamepad.current.rightStick).ReadValue();
		}
		int num = _wheel.CurrentSliceFromAxis(axis);
		_wheel.SetHovered(num);
		if (num != _wheelHoverIndex)
		{
			_wheelHoverIndex = num;
			_wheelHoverStart = ((num >= 0) ? Time.time : (-1f));
		}
		else if (num >= 0 && _wheelHoverStart > 0f && Time.time - _wheelHoverStart >= 0.13f)
		{
			_wheelHoverStart = -1f;
			if (ActivateWheelSelection(num))
			{
				ShowWheel(show: false);
			}
		}
	}

	private bool ActivateWheelSelection(int idx)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_0352: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)_wheel == (Object)null)
		{
			return false;
		}
		RadialMenu3D.Item item = _wheel.GetItem(idx);
		if (item == null)
		{
			return false;
		}
		switch (_wheelPage)
		{
		case WheelPage.Root:
			switch (item.Id)
			{
			case "toggles":
				BuildWheelPage(_wheelPage = WheelPage.Toggles);
				return false;
			case "teleport":
				BuildWheelPage(_wheelPage = WheelPage.Teleport);
				return false;
			case "speed":
				BuildWheelPage(_wheelPage = WheelPage.Speed);
				return false;
			case "close":
				return true;
			}
			break;
		case WheelPage.Toggles:
			switch (item.Id)
			{
			case "bh":
				BHToggle();
				_wheel.RefreshLabels();
				return false;
			case "cs":
				CSToggle();
				_wheel.RefreshLabels();
				return false;
			case "smt":
				SmiteToggle();
				_wheel.RefreshLabels();
				return false;
			case "dev":
				Actions.toggle_devlight();
				_wheel.RefreshLabels();
				return false;
			case "nv":
				Actions.no_void_tp = !Actions.no_void_tp;
				_wheel.RefreshLabels();
				return false;
			case "rev":
				Actions.revive();
				return false;
			case "ar":
				SetAutoReviveEnabled(!_autoReviveEnabled);
				_wheel.RefreshLabels();
				return false;
			case "mf":
				_mapFollowEnabled = !_mapFollowEnabled;
				_wheel.RefreshLabels();
				return false;
			case "back":
				BuildWheelPage(_wheelPage = WheelPage.Root);
				return false;
			}
			break;
		case WheelPage.Teleport:
		{
			if (item.Id == "tp-prev")
			{
				_tpPageIndex = Mathf.Max(0, _tpPageIndex - 1);
				BuildWheelPage(WheelPage.Teleport);
				return false;
			}
			if (item.Id == "tp-next")
			{
				_tpPageIndex++;
				BuildWheelPage(WheelPage.Teleport);
				return false;
			}
			if (item.Id == "back")
			{
				BuildWheelPage(_wheelPage = WheelPage.Root);
				return false;
			}
			if (_teleports.TryGetValue(item.Id, out var value))
			{
				PlayerController.Current.LocomotionController.MoveTo(value, (LocomotionFunction)0);
				return true;
			}
			break;
		}
		case WheelPage.Speed:
		{
			if (item.Id == "back")
			{
				BuildWheelPage(_wheelPage = WheelPage.Root);
				return false;
			}
			if (!item.Id.StartsWith("spd_"))
			{
				break;
			}
			if (int.TryParse(item.Id.Substring(4), out var result))
			{
				float num = (_currentSpeedCache = Mathf.Lerp(0.2f, 15f, (float)result / 9f));
				Actions.set_speed(num);
				PlayerController current = PlayerController.Current;
				if ((Object)(object)current != (Object)null)
				{
					PlayerMessageDisplay messageDisplay = current.MessageDisplay;
					if ((Object)(object)messageDisplay != (Object)null)
					{
						messageDisplay.Display($"Speed: {num:F2}", 1.2f, (DisplayMessageType)2);
					}
				}
				_wheel.RefreshLabels();
			}
			return false;
		}
		}
		return false;
	}

	private void BuildWheelPage(WheelPage p)
	{
		EnsureWheel();
		List<RadialMenu3D.Item> list = new List<RadialMenu3D.Item>();
		switch (p)
		{
		case WheelPage.Root:
			list.Add(new RadialMenu3D.Item("toggles", "Toggles"));
			list.Add(new RadialMenu3D.Item("teleport", "Teleport"));
			list.Add(new RadialMenu3D.Item("speed", "Speed"));
			list.Add(new RadialMenu3D.Item("map", "Map"));
			list.Add(new RadialMenu3D.Item("close", "Close"));
			_wheel.SetItems(list, GetWheelTitle("Main"));
			break;
		case WheelPage.Toggles:
			list.Add(new RadialMenu3D.Item("bh", () => _bhEnabled ? "Blackhole: ON" : "Blackhole: OFF"));
			list.Add(new RadialMenu3D.Item("cs", () => _csEnabled ? "Cruel: ON" : "Cruel: OFF"));
			list.Add(new RadialMenu3D.Item("smt", () => _smiteEnabled ? "Smite: ON" : "Smite: OFF"));
			list.Add(new RadialMenu3D.Item("dev", () => Actions.is_devlight ? "Dev Light: ON" : "Dev Light: OFF"));
			list.Add(new RadialMenu3D.Item("nv", () => Actions.no_void_tp ? "No-void TP: ON" : "No-void TP: OFF"));
			list.Add(new RadialMenu3D.Item("rev", "Revive"));
			list.Add(new RadialMenu3D.Item("ar", () => _autoReviveEnabled ? "Auto Revive: ON" : "Auto Revive: OFF"));
			list.Add(new RadialMenu3D.Item("mf", () => _mapFollowEnabled ? "3D Map: ON" : "3D Map: OFF"));
			list.Add(new RadialMenu3D.Item("back", "Back"));
			_wheel.SetItems(list, GetWheelTitle("Toggles"));
			break;
		case WheelPage.Speed:
		{
			for (int j = 0; j < 10; j++)
			{
				int k = j;
				list.Add(new RadialMenu3D.Item($"spd_{k}", delegate
				{
					float num3 = Mathf.Lerp(0.2f, 15f, (float)k / 9f);
					return (Mathf.Abs(((_currentSpeedCache < 0f) ? ReadBaseSpeed() : _currentSpeedCache) - num3) < 0.01f) ? $"★ {num3:F1}" : $"{num3:F1}";
				}));
			}
			list.Add(new RadialMenu3D.Item("back", "Back"));
			_wheel.SetItems(list, GetWheelTitle("Speed"));
			break;
		}
		case WheelPage.Teleport:
		{
			List<string> list2 = _teleports.Keys.ToList();
			int num = _tpPageIndex * 5;
			if (num >= list2.Count)
			{
				_tpPageIndex = 0;
				num = 0;
			}
			int num2 = Mathf.Min(num + 5, list2.Count);
			for (int i = num; i < num2; i++)
			{
				string text = list2[i];
				list.Add(new RadialMenu3D.Item(text, text));
			}
			list.Add(new RadialMenu3D.Item("tp-prev", "Prev"));
			list.Add(new RadialMenu3D.Item("tp-next", "Next"));
			list.Add(new RadialMenu3D.Item("back", "Back"));
			_wheel.SetItems(list, GetWheelTitle($"Teleport ({num + 1}-{num2}/{list2.Count})"));
			break;
		}
		}
	}

	private void CSSetEnabled(bool on)
	{
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Expected O, but got Unknown
		if (_csEnabled == on)
		{
			return;
		}
		_csEnabled = on;
		if (_csEnabled)
		{
			if ((Object)Actions.find_hand(right: true) != (Object)null)
			{
				CSRuntime.SpawnForHand(Actions.find_hand(right: true));
				PlayerController current = PlayerController.Current;
				Camera val = (((Object)(object)current != (Object)null) ? current.Camera : null) ?? Camera.main;
				CSRuntime.SetViewTransform(((Object)(object)val != (Object)null) ? ((Component)val).transform : null);
				CSRuntime.SetSmiteEnabled(_smiteEnabled);
				CSRuntime.SetBladeHeldFxEnabled(_bladeHeldFxEnabled);
			}
		}
		else
		{
			CSRuntime.SetSmiteEnabled(_smiteEnabled);
			CSRuntime.SetBladeHeldFxEnabled(_bladeHeldFxEnabled);
		}
	}

	private void CSToggle()
	{
		CSSetEnabled(!_csEnabled);
	}

	private void SmiteSetEnabled(bool on)
	{
		if (_smiteEnabled != on)
		{
			_smiteEnabled = on;
			CSRuntime.SetSmiteEnabled(on);
		}
	}

	private void BladeHeldFxSetEnabled(bool on)
	{
		if (_bladeHeldFxEnabled != on)
		{
			_bladeHeldFxEnabled = on;
			CSRuntime.SetBladeHeldFxEnabled(on);
		}
	}

	private void SmiteToggle()
	{
		SmiteSetEnabled(!_smiteEnabled);
	}

	private void BladeHeldFxToggle()
	{
		BladeHeldFxSetEnabled(!_bladeHeldFxEnabled);
	}

	private string GetWheelTitle(string s)
	{
		return "< " + s + " >";
	}

	private static bool IsPickupHeld(Pickup p)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)p == (Object)null)
		{
			return false;
		}
		Type type = ((object)p).GetType();
		PropertyInfo property = type.GetProperty("IsHeld", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (property != null && property.PropertyType == typeof(bool))
		{
			try
			{
				return (bool)property.GetValue(p);
			}
			catch
			{
			}
		}
		PropertyInfo property2 = type.GetProperty("Holder", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (property2 != null)
		{
			try
			{
				if (property2.GetValue(p) != null)
				{
					return true;
				}
			}
			catch
			{
			}
		}
		FieldInfo field = type.GetField("holder", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (field != null)
		{
			try
			{
				if (field.GetValue(p) != null)
				{
					return true;
				}
			}
			catch
			{
			}
		}
		string text = (((Object)(object)((Component)p).transform.parent != (Object)null) ? ((Object)((Component)p).transform.parent).name.ToLowerInvariant() : "");
		if (text.Contains("hand") || text.Contains("interactor"))
		{
			return true;
		}
		return false;
	}

	private void ClearSavedData()
	{
		try
		{
			if (!string.IsNullOrEmpty(_savePath) && File.Exists(_savePath))
			{
				File.Delete(_savePath);
				MelonLogger.Msg("[Unified] Deleted save file: " + _savePath);
			}
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] ClearSavedData delete failed: " + ex.GetBaseException().Message);
		}
		_save = new SaveBlob();
		_saved.Clear();
		RefreshJoinableServerIds();
		_tpOptions = null;
		_selServer = -1;
		_srvId = string.Empty;
		_srvName = string.Empty;
		_statusLeft = "Saved data cleared.";
	}

	private async Task get_things()
	{
		await Task.Delay(500);
if (false)
{
    return;
}
		Basemodal.main_values.is_loaded_in = true;
		Actions.master_terrain = GameObject.Find(Basemodal.main_values.pc2q ? "MasterTerrain" : "MasterTerrain_Test").transform;
		Actions.base_terrain_pos = Actions.master_terrain.position;
		await Basemodal.main_values.api_client.send_to_ws($"event joined-server {Basemodal.main_values.target_server_id}");
		await Basemodal.main_values.api_client.ConnectPhysicsServer(Basemodal.main_values.target_server_id);
		try
		{
			BasemodalApiInteractor api_client = Basemodal.main_values.api_client;
			BasemodalApiInteractor basemodalApiInteractor = api_client;
			OrbUtils.spawn_orb(await basemodalApiInteractor.GetOrb((await Basemodal.main_values.api_client.GetMe()).username), ((UserInfo)ApiAccess.ApiClient.UserClient.LoggedInUserInfo).Identifier);
		}
		catch (Exception arg)
		{
			MelonLogger.Error($"unable to spawn orb {arg}");
		}
	}

	private void DrawPrefabulatorTopBar()
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_002e: Unknown result type (might be due to invalid IL or missing references)
		//IL_05b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_05c0: Expected O, but got Unknown
		//IL_070f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0714: Unknown result type (might be due to invalid IL or missing references)
		//IL_08ca: Unknown result type (might be due to invalid IL or missing references)
		//IL_08f5: Unknown result type (might be due to invalid IL or missing references)
		//IL_0903: Unknown result type (might be due to invalid IL or missing references)
		//IL_0947: Unknown result type (might be due to invalid IL or missing references)
		//IL_094a: Unknown result type (might be due to invalid IL or missing references)
		//IL_094f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0953: Unknown result type (might be due to invalid IL or missing references)
		//IL_0958: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a20: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a26: Invalid comparison between Unknown and I4
		//IL_0a3a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a41: Invalid comparison between Unknown and I4
		//IL_0a2d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a33: Invalid comparison between Unknown and I4
		//IL_0982: Unknown result type (might be due to invalid IL or missing references)
		//IL_09d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a48: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a52: Invalid comparison between Unknown and I4
		//IL_09e5: Unknown result type (might be due to invalid IL or missing references)
		Rect val = default(Rect);
		((Rect)(ref val))._002Ector(0f, 0f, (float)Screen.width, 68f);
		Rect val2 = default(Rect);
		bool flag = false;
		GUILayout.BeginArea(val, _card);
		GUILayout.BeginVertical(Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		bool pfv3EditorOn = _pfv3EditorOn;
		_pfv3EditorOn = GUILayout.Toggle(_pfv3EditorOn, _pfv3EditorOn ? "Editor: ON" : "Editor: OFF", _topTab, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(28f),
			GUILayout.Width(110f)
		});
		if (_pfv3EditorOn != pfv3EditorOn)
		{
			EnsureEditorCameraActive(_pfv3EditorOn);
		}
		if (GUILayout.Toggle(_pfv3Mode == PFV3Mode.Translate, "Move (W)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(90f)
		}))
		{
			_pfv3Mode = PFV3Mode.Translate;
		}
		if (GUILayout.Toggle(_pfv3Mode == PFV3Mode.Rotate, "Rotate (E)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(90f)
		}))
		{
			_pfv3Mode = PFV3Mode.Rotate;
		}
		if (GUILayout.Toggle(_pfv3Mode == PFV3Mode.Scale, "Scale (R)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(90f)
		}))
		{
			_pfv3Mode = PFV3Mode.Scale;
		}
		GUILayout.Space(8f);
		GUILayout.Label("Snap Δ", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(42f) });
		GUILayout.Label("Move", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(34f) });
		_pfv3MoveSnap = Mathf.Max(0.01f, PFV3_FloatFieldInline(_pfv3MoveSnap, 46f));
		GUILayout.Space(4f);
		GUILayout.Label("Rot", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(24f) });
		_pfv3RotateSnap = Mathf.Round(PFV3_FloatFieldInline(_pfv3RotateSnap, 46f));
		GUILayout.Space(4f);
		GUILayout.Label("Scale", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(40f) });
		_pfv3ScaleSnap = Mathf.Max(0.01f, PFV3_FloatFieldInline(_pfv3ScaleSnap, 46f));
		GUILayout.Space(10f);
		if (GUILayout.Button(_pfv3ScanDrop ? "Scan ▼" : "Scan ▸", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(78f)
		}))
		{
			_pfv3ScanDrop = !_pfv3ScanDrop;
		}
		string text = ((_pfv3Sel.Items.Count == 0) ? "Selected: (none)" : ((_pfv3SelFocus >= 0 && _pfv3SelFocus < _pfv3Sel.Items.Count) ? ("Selected: " + _pfv3Sel.Items[_pfv3SelFocus].Name) : $"Selected: {_pfv3Sel.Items.Count}"));
		if (GUILayout.Button(_pfv3SelDrop ? text.Replace("▸", "▼") : (text + " ▸"), _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.MinWidth(160f)
		}))
		{
			_pfv3SelDrop = !_pfv3SelDrop;
		}
		GUILayout.FlexibleSpace();
		if (GUILayout.Button("Scan", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(80f)
		}))
		{
			PFV3_DoScan();
		}
		if (GUILayout.Button("Confirm", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(90f)
		}))
		{
			PFV3_ConfirmToServer();
		}
		if (GUILayout.Button("Undo", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(70f)
		}))
		{
			PFV3_Undo();
		}
		if (GUILayout.Button("Redo", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(70f)
		}))
		{
			PFV3_Redo();
		}
		if (GUILayout.Button("Clear", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(26f),
			GUILayout.Width(70f)
		}))
		{
			PFV3_Reset();
		}
		GUILayout.Space(8f);
		GUILayout.Label("Console:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
		GUI.SetNextControlName("PFV3Console");
		_pfv3CmdLine = GUILayout.TextField(_pfv3CmdLine ?? "", (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.MinWidth(260f),
			GUILayout.Height(24f)
		});
		GUILayout.Space(6f);
		if ((Object)_pfv3EditorCtl != (Object)null)
		{
			bool lookFrozen = _pfv3EditorCtl.lookFrozen;
			bool flag2 = GUILayout.Toggle(lookFrozen, "Mouse-free (-)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
			{
				GUILayout.Height(24f),
				GUILayout.Width(130f)
			});
			if (flag2 != lookFrozen)
			{
				_pfv3EditorCtl.lookFrozen = flag2;
				if (flag2)
				{
					Cursor.lockState = (CursorLockMode)0;
					Cursor.visible = true;
				}
				else
				{
					Cursor.lockState = (CursorLockMode)1;
					Cursor.visible = false;
				}
			}
		}
		GUILayout.EndHorizontal();
		GUILayout.Space(2f);
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.FlexibleSpace();
		GUILayout.Label("File:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(32f) });
		if (_pfv3TxtFiles == null || _pfv3TxtFiles.Length == 0)
		{
			PFV3_RefreshTxtList();
		}
		if (GUILayout.Button((_pfv3TxtIndex >= 0 && _pfv3TxtIndex < _pfv3TxtFiles.Length) ? Path.GetFileName(_pfv3TxtFiles[_pfv3TxtIndex]) : "(no .txt)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(22f),
			GUILayout.Width(160f)
		}))
		{
			PFV3_RefreshTxtList();
			_pfv3ShowFileMenu = !_pfv3ShowFileMenu;
		}
		Rect lastRect = GUILayoutUtility.GetLastRect();
		((Rect)(ref val2))._002Ector(((Rect)(ref val)).x + ((Rect)(ref lastRect)).x, ((Rect)(ref val)).y + ((Rect)(ref lastRect)).y, ((Rect)(ref lastRect)).width, ((Rect)(ref lastRect)).height);
		flag = true;
		GUI.enabled = _pfv3TxtFiles != null && _pfv3TxtFiles.Length != 0;
		if (GUILayout.Button(_pfv3SpawnPreviewActive ? "Spawn (server)" : "Spawn (preview)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(22f),
			GUILayout.Width(120f)
		}))
		{
			if (!_pfv3SpawnPreviewActive)
			{
				if (_pfv3TxtIndex >= 0 && _pfv3TxtIndex < _pfv3TxtFiles.Length)
				{
					PFV3_LoadTxtIntoPreview(_pfv3TxtFiles[_pfv3TxtIndex]);
				}
			}
			else
			{
				PFV3_ConfirmSpawnFromPreview();
			}
		}
		GUI.enabled = true;
		if (GUILayout.Button("Save", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(22f),
			GUILayout.Width(56f)
		}))
		{
			PFV3_SaveToText(_pfv3SaveNearby, _pfv3SaveName);
		}
		GUILayout.Space(12f);
		GUILayout.FlexibleSpace();
		GUILayout.Label("Name:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(42f) });
		_pfv3SaveName = GUILayout.TextField(_pfv3SaveName ?? "", (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(22f),
			GUILayout.Width(140f)
		});
		GUILayout.EndHorizontal();
		GUILayout.EndVertical();
		GUILayout.EndArea();
		if (_pfv3ShowFileMenu && flag)
		{
			Rect val3 = default(Rect);
			((Rect)(ref val3))._002Ector(((Rect)(ref val2)).x, ((Rect)(ref val2)).yMax + 2f, 240f, 200f);
			GUI.Box(val3, GUIContent.none);
			Rect val4 = default(Rect);
			((Rect)(ref val4))._002Ector(0f, 0f, ((Rect)(ref val3)).width - 16f, Mathf.Max(200f, (float)_pfv3TxtFiles.Length * 22f + 8f));
			_pfv3FileMenuScroll = GUI.BeginScrollView(val3, _pfv3FileMenuScroll, val4, false, true);
			float num = 4f;
			for (int i = 0; i < _pfv3TxtFiles.Length; i++)
			{
				if (GUI.Button(new Rect(4f, num, ((Rect)(ref val4)).width - 8f, 20f), Path.GetFileName(_pfv3TxtFiles[i]), _sidebarBtn))
				{
					_pfv3TxtIndex = i;
					_pfv3ShowFileMenu = false;
				}
				num += 22f;
			}
			GUI.EndScrollView();
			if ((int)Event.current.type == 0 && !((Rect)(ref val3)).Contains(Event.current.mousePosition))
			{
				_pfv3ShowFileMenu = false;
				Event.current.Use();
			}
		}
		PFV3_DrawRightPanel();
		PFV3_HandleInput();
		PFV3_DrawOverlays();
		if (Event.current != null && ((int)Event.current.rawType == 4 || (int)Event.current.rawType == 5) && ((int)Event.current.keyCode == 13 || (int)Event.current.keyCode == 271) && GUI.GetNameOfFocusedControl() == "PFV3Console")
		{
			if (!string.IsNullOrWhiteSpace(_pfv3CmdLine))
			{
				SendConsoleCmd(_pfv3CmdLine);
			}
			_pfv3CmdLine = "";
			Event.current.Use();
			GUI.FocusControl((string)null);
		}
	}

	private async Task delay_set_kicked()
	{
		try
		{
			await Task.Delay(250);
			try
			{
				Pc2qPatch.Con = null;
			}
			catch
			{
			}
			PreJoinResult = null;
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[Unified] delay_set_kicked error: " + ex.GetBaseException().Message);
		}
	}

	private void SendConsoleCmd(string cmd)
	{
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Expected O, but got Unknown
		RuntimeConsole val = Resources.FindObjectsOfTypeAll<RuntimeConsole>().FirstOrDefault();
		if ((Object)val == (Object)null)
		{
			MelonLogger.Warning("[Unified] RuntimeConsole not found.");
			return;
		}
		MethodInfo method = ((object)val).GetType().GetMethod("RunCommandOnServer", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
		if (method == null)
		{
			MelonLogger.Warning("[Unified] RunCommandOnServer not found.");
			return;
		}
		method.Invoke(val, new object[1] { cmd });
		MelonLogger.Msg("[Unified] Executed: " + cmd);
	}

	private void PFV3_DeleteSelected()
	{
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (!string.IsNullOrEmpty(item.Id))
			{
				SendConsoleCmd("select " + item.Id);
				SendConsoleCmd("select destroy");
			}
		}
	}

	private void PFV3_SnapGroundSelected()
	{
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (!string.IsNullOrEmpty(item.Id))
			{
				SendConsoleCmd("select " + item.Id);
				SendConsoleCmd("select snap-ground");
			}
		}
	}

	private void PFV3_SnapToSelected(string targetIdRaw)
	{
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		Match match = Regex.Match(targetIdRaw ?? "", "^\\s*(\\d+)");
		string text = (match.Success ? match.Groups[1].Value : null);
		if (string.IsNullOrEmpty(text))
		{
			return;
		}
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (!string.IsNullOrEmpty(item.Id))
			{
				SendConsoleCmd("select " + item.Id);
				SendConsoleCmd("Select snap-to " + text);
			}
		}
	}

	private void PFV3_SelectLookAtMe()
	{
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		IHighLevelApiClient apiClient = ApiAccess.ApiClient;
		object obj;
		if (apiClient == null)
		{
			obj = null;
		}
		else
		{
			IUserApiClient userClient = apiClient.UserClient;
			if (userClient == null)
			{
				obj = null;
			}
			else
			{
				UserInfoWithPermissions loggedInUserInfo = userClient.LoggedInUserInfo;
				obj = ((loggedInUserInfo != null) ? ((UserInfo)loggedInUserInfo).Username : null);
			}
		}
		if (obj == null)
		{
			PlayerController current = PlayerController.Current;
			obj = (((Object)(object)current != (Object)null) ? ((Object)current).name : null) ?? "me";
		}
		string text = (string)obj;
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (!string.IsNullOrEmpty(item.Id))
			{
				SendConsoleCmd("select " + item.Id);
				SendConsoleCmd("select look-at " + text);
			}
		}
	}

	private void PFV3_UnselectAll()
	{
		_pfv3Sel.Clear();
		_pfv3SelFocus = -1;
		SendConsoleCmd("Select unselect");
	}

	private static void PFV3_EnsureLineMat()
	{
		//IL_0005: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Expected O, but got Unknown
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_002e: Expected O, but got Unknown
		if (!((Object)_pfv3LineMat != (Object)null))
		{
			_pfv3LineMat = new Material(Shader.Find("Hidden/Internal-Colored"))
			{
				hideFlags = (HideFlags)61
			};
			_pfv3LineMat.SetInt("_SrcBlend", 5);
			_pfv3LineMat.SetInt("_DstBlend", 10);
			_pfv3LineMat.SetInt("_Cull", 0);
			_pfv3LineMat.SetInt("_ZWrite", 0);
		}
	}

	private static Vector3 PFV3_AxisDir(int axis)
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		return (Vector3)(axis switch
		{
			1 => Vector3.up, 
			0 => Vector3.right, 
			_ => Vector3.forward, 
		});
	}

	private Vector2 PFV3_Project(Vector3 world)
	{
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return Vector2.zero;
		}
		Vector3 val = pFV3_Cam.WorldToScreenPoint(world);
		return new Vector2(val.x, val.y);
	}

	private static float PFV3_SqrDistPointSegment(Vector2 p, Vector2 a, Vector2 b)
	{
		//IL_0000: Unknown result type (might be due to invalid IL or missing references)
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0008: Unknown result type (might be due to invalid IL or missing references)
		//IL_0009: Unknown result type (might be due to invalid IL or missing references)
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_002f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0037: Unknown result type (might be due to invalid IL or missing references)
		//IL_003c: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		Vector2 val = b - a;
		float num = Vector2.Dot(p - a, val) / Mathf.Max(1E-06f, ((Vector2)(ref val)).sqrMagnitude);
		num = Mathf.Clamp01(num);
		Vector2 val2 = a + val * num;
		Vector2 val3 = p - val2;
		return ((Vector2)(ref val3)).sqrMagnitude;
	}

	private int PFV3_IntField(int v, int min, int max)
	{
		int.TryParse(GUILayout.TextField(v.ToString(CultureInfo.InvariantCulture), (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) }), out v);
		return Mathf.Clamp(v, min, max);
	}

	private float PFV3_FloatField(float v)
	{
		float.TryParse(GUILayout.TextField(v.ToString("0.###", CultureInfo.InvariantCulture), (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) }), NumberStyles.Float, CultureInfo.InvariantCulture, out v);
		return v;
	}

	private void PFV3_SelectSingle(PFV3Captured c)
	{
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Expected O, but got Unknown
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Expected O, but got Unknown
		//IL_010e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0113: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_011f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0126: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ca: Unknown result type (might be due to invalid IL or missing references)
		//IL_013d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0142: Unknown result type (might be due to invalid IL or missing references)
		//IL_0148: Unknown result type (might be due to invalid IL or missing references)
		//IL_014d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0153: Unknown result type (might be due to invalid IL or missing references)
		//IL_0158: Unknown result type (might be due to invalid IL or missing references)
		//IL_015f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0164: Unknown result type (might be due to invalid IL or missing references)
		_pfv3Sel.Items.Clear();
		_pfv3SelFocus = -1;
		if (c == null)
		{
			_pfv3Sel.Phase = PFV3SelPhase.Idle;
			return;
		}
		Transform val = (((Object)c.Tr != (Object)null) ? c.Tr : PFV3_FindTransformFor(c));
		string id = PFV3_LeadingDigits(c.Display) ?? c.Id ?? c.Hash.ToString();
		if ((Object)val != (Object)null)
		{
			_pfv3Sel.Items.Add(new PFV3Captured
			{
				Display = c.Display,
				Id = id,
				Hash = c.Hash,
				Pos = val.position,
				Rot = val.rotation,
				Scale = val.localScale,
				Tr = val
			});
		}
		else
		{
			_pfv3Sel.Items.Add(new PFV3Captured
			{
				Display = c.Display,
				Id = id,
				Hash = c.Hash,
				Pos = c.Pos,
				Rot = c.Rot,
				Scale = c.Scale,
				Tr = null
			});
		}
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
		_pfv3Pivot = PFV3_ComputePivot();
		_pfv3Sel.Phase = PFV3SelPhase.Selected;
		_pfv3SelFocus = 0;
		_pfv3Sel.Active = _pfv3Sel.Items.Count > 0;
		_pfv3HaveEulerSent = false;
		SendConsoleCmd("select " + c.Id);
	}

	private static Transform PFV3_FindTransformFor(PFV3Captured c)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Unknown result type (might be due to invalid IL or missing references)
		//IL_0056: Unknown result type (might be due to invalid IL or missing references)
		//IL_005b: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)c.Tr != (Object)null)
		{
			return c.Tr;
		}
		Transform result = null;
		float num = float.PositiveInfinity;
		NetworkPrefab[] array = Object.FindObjectsOfType<NetworkPrefab>();
		foreach (NetworkPrefab val in array)
		{
			if (val.Hash == c.Hash)
			{
				Transform transform = ((Component)val).transform;
				Vector3 val2 = transform.position - c.Pos;
				float sqrMagnitude = ((Vector3)(ref val2)).sqrMagnitude;
				if (sqrMagnitude < num)
				{
					num = sqrMagnitude;
					result = transform;
				}
			}
		}
		return result;
	}

	private void PFV3_Tick()
	{
		//IL_0009: Unknown result type (might be due to invalid IL or missing references)
		//IL_000e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0072: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_0134: Unknown result type (might be due to invalid IL or missing references)
		//IL_013a: Invalid comparison between Unknown and I4
		//IL_02a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ab: Invalid comparison between Unknown and I4
		//IL_0141: Unknown result type (might be due to invalid IL or missing references)
		//IL_0148: Invalid comparison between Unknown and I4
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_011f: Unknown result type (might be due to invalid IL or missing references)
		//IL_02af: Unknown result type (might be due to invalid IL or missing references)
		//IL_02b6: Invalid comparison between Unknown and I4
		//IL_015a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0161: Invalid comparison between Unknown and I4
		//IL_02ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c1: Invalid comparison between Unknown and I4
		//IL_0173: Unknown result type (might be due to invalid IL or missing references)
		//IL_017a: Invalid comparison between Unknown and I4
		//IL_02c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_02cc: Invalid comparison between Unknown and I4
		//IL_018c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0548: Unknown result type (might be due to invalid IL or missing references)
		//IL_0216: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ac: Invalid comparison between Unknown and I4
		//IL_0332: Unknown result type (might be due to invalid IL or missing references)
		//IL_022f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0236: Invalid comparison between Unknown and I4
		//IL_0563: Unknown result type (might be due to invalid IL or missing references)
		//IL_05e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_0352: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_024a: Unknown result type (might be due to invalid IL or missing references)
		//IL_024f: Unknown result type (might be due to invalid IL or missing references)
		//IL_07ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_05aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_0598: Unknown result type (might be due to invalid IL or missing references)
		//IL_05af: Unknown result type (might be due to invalid IL or missing references)
		//IL_05b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_05ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_05c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_05c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_05cb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f6: Unknown result type (might be due to invalid IL or missing references)
		//IL_01fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01fd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0202: Unknown result type (might be due to invalid IL or missing references)
		//IL_0862: Unknown result type (might be due to invalid IL or missing references)
		//IL_0867: Unknown result type (might be due to invalid IL or missing references)
		//IL_086a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0281: Unknown result type (might be due to invalid IL or missing references)
		//IL_0286: Unknown result type (might be due to invalid IL or missing references)
		//IL_0288: Unknown result type (might be due to invalid IL or missing references)
		//IL_028d: Unknown result type (might be due to invalid IL or missing references)
		//IL_087d: Unknown result type (might be due to invalid IL or missing references)
		//IL_08a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_08aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_08b6: Unknown result type (might be due to invalid IL or missing references)
		//IL_08bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_08c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_08c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_08e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_08ee: Unknown result type (might be due to invalid IL or missing references)
		//IL_06b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_06b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_06bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_06c2: Unknown result type (might be due to invalid IL or missing references)
		//IL_06ca: Unknown result type (might be due to invalid IL or missing references)
		//IL_06cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_03c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_090c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0913: Unknown result type (might be due to invalid IL or missing references)
		//IL_091a: Unknown result type (might be due to invalid IL or missing references)
		//IL_091f: Unknown result type (might be due to invalid IL or missing references)
		//IL_03ee: Unknown result type (might be due to invalid IL or missing references)
		//IL_03dc: Unknown result type (might be due to invalid IL or missing references)
		//IL_03f3: Unknown result type (might be due to invalid IL or missing references)
		//IL_03f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_03fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_0405: Unknown result type (might be due to invalid IL or missing references)
		//IL_0411: Unknown result type (might be due to invalid IL or missing references)
		//IL_0416: Unknown result type (might be due to invalid IL or missing references)
		//IL_041b: Unknown result type (might be due to invalid IL or missing references)
		//IL_074f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0754: Unknown result type (might be due to invalid IL or missing references)
		//IL_04f4: Unknown result type (might be due to invalid IL or missing references)
		//IL_04f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_050e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0518: Unknown result type (might be due to invalid IL or missing references)
		//IL_0527: Unknown result type (might be due to invalid IL or missing references)
		//IL_052c: Unknown result type (might be due to invalid IL or missing references)
		//IL_046d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0472: Unknown result type (might be due to invalid IL or missing references)
		//IL_04b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_04b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_04ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_04bf: Unknown result type (might be due to invalid IL or missing references)
		if (!_pfv3EditorOn)
		{
			return;
		}
		Vector2 val = PFV3_MouseScreen();
		Vector3 val2 = default(Vector3);
		((Vector3)(ref val2))._002Ector(val.x, val.y, 0f);
		if (PFV3_Block3DInput(val))
		{
			_pfv3HoverName = null;
			_pfv3HoverHash = 0u;
			return;
		}
		bool num = val2.y >= (float)Screen.height - 68f;
		bool flag = val2.x >= (float)Screen.width - 360f && val2.y <= (float)Screen.height - 68f;
		if (num || flag)
		{
			_pfv3HoverName = null;
			_pfv3HoverHash = 0u;
			return;
		}
		if ((Object)(object)PFV3_Cam == (Object)null)
		{
			return;
		}
		_pfv3HoverName = null;
		_pfv3HoverHash = 0u;
		_pfv3HoverTr = null;
		if (PFV3_RaycastNP(val, out var np, out var bestHit))
		{
			_pfv3HoverHash = np.Hash;
			_pfv3HoverName = (_pfv3Hash2Name.TryGetValue(np.Hash, out var value) ? value : ((Object)((Component)np).gameObject).name);
			_pfv3HoverTr = ((Component)np).transform;
			_pfv3HoverPos = _pfv3HoverTr.position;
		}
		Event current = Event.current;
		if (current != null && (int)current.type == 4)
		{
			if ((int)current.keyCode == 120)
			{
				_pfv3KeyAxis = 0;
				current.Use();
			}
			if ((int)current.keyCode == 121)
			{
				_pfv3KeyAxis = 1;
				current.Use();
			}
			if ((int)current.keyCode == 122)
			{
				_pfv3KeyAxis = 2;
				current.Use();
			}
			if (((Enum)current.modifiers).HasFlag((Enum)(object)(EventModifiers)2) && (int)current.keyCode == 114)
			{
				int num2 = ((_pfv3KeyAxis < 0) ? 1 : _pfv3KeyAxis);
				float pfv3RotateSnap = _pfv3RotateSnap;
				Vector3 zero = Vector3.zero;
				if (num2 == 0)
				{
					zero.x = pfv3RotateSnap;
				}
				if (num2 == 1)
				{
					zero.y = pfv3RotateSnap;
				}
				if (num2 == 2)
				{
					zero.z = pfv3RotateSnap;
				}
				_pfv3PreviewEuler += zero;
				PFV3_ApplyPreviewLocally();
				current.Use();
			}
			if (((Enum)current.modifiers).HasFlag((Enum)(object)(EventModifiers)2) && (int)current.keyCode == 102)
			{
				int num3 = ((_pfv3KeyAxis < 0) ? 1 : _pfv3KeyAxis);
				Vector3 zero2 = Vector3.zero;
				if (num3 == 0)
				{
					zero2.x = 180f;
				}
				if (num3 == 1)
				{
					zero2.y = 180f;
				}
				if (num3 == 2)
				{
					zero2.z = 180f;
				}
				_pfv3PreviewEuler += zero2;
				PFV3_ApplyPreviewLocally();
				current.Use();
			}
		}
		if (current != null && (int)current.type == 5 && ((int)current.keyCode == 120 || (int)current.keyCode == 121 || (int)current.keyCode == 122))
		{
			_pfv3KeyAxis = -1;
			current.Use();
		}
		float num4 = PFV3_ReadScrollY();
		bool flag2 = _pfv3Sel != null && _pfv3Sel.Items.Count > 0 && _pfv3Sel.Phase >= PFV3SelPhase.Selected;
		if (flag2 && Mathf.Abs(num4) > 0.001f)
		{
			bool flag3 = current != null && ((Enum)current.modifiers).HasFlag((Enum)(object)(EventModifiers)2);
			bool flag4 = current != null && ((Enum)current.modifiers).HasFlag((Enum)(object)(EventModifiers)1);
			float num5 = (flag3 ? _pfv3MoveSnap : (flag4 ? (_pfv3NudgeStep * 0.1f) : _pfv3NudgeStep));
			int num6 = ((_pfv3KeyAxis >= 0) ? _pfv3KeyAxis : ((_pfv3ActiveAxis >= 0) ? _pfv3ActiveAxis : (-1)));
			if (_pfv3Mode == PFV3Mode.Translate)
			{
				Vector3 val3 = ((num6 >= 0) ? PFV3_GetAxisDir(num6) : (((Object)(object)PFV3_Cam != (Object)null) ? ((Component)PFV3_Cam).transform.forward : Vector3.forward));
				_pfv3PreviewOffset += ((Vector3)(ref val3)).normalized * num5 * Mathf.Sign(num4);
				PFV3_ApplyPreviewLocally();
			}
			else if (_pfv3Mode == PFV3Mode.Rotate)
			{
				int num7 = ((num6 < 0) ? 1 : num6);
				float num8 = (flag3 ? _pfv3RotateSnap : (flag4 ? (_pfv3RotateSnap * 0.1f) : (_pfv3RotateSnap * 0.25f)));
				Vector3 zero3 = Vector3.zero;
				if (num7 == 0)
				{
					zero3.x = num8 * Mathf.Sign(num4);
				}
				if (num7 == 1)
				{
					zero3.y = num8 * Mathf.Sign(num4);
				}
				if (num7 == 2)
				{
					zero3.z = num8 * Mathf.Sign(num4);
				}
				_pfv3PreviewEuler += zero3;
				PFV3_ApplyPreviewLocally();
			}
			else if (_pfv3Mode == PFV3Mode.Scale)
			{
				float num9 = (flag3 ? _pfv3ScaleSnap : (flag4 ? 0.01f : 0.05f));
				Vector3 pfv3PreviewScale = _pfv3PreviewScale;
				float num10 = 1f + num9 * Mathf.Sign(num4);
				_pfv3PreviewScale = Vector3.one * Mathf.Max(0.001f, pfv3PreviewScale.x * num10);
				PFV3_ApplyPreviewLocally();
			}
			if (current != null)
			{
				current.Use();
			}
		}
		if (current != null && (int)current.type == 0 && current.button == 2 && flag2)
		{
			float num11 = (((Enum)current.modifiers).HasFlag((Enum)(object)(EventModifiers)2) ? _pfv3MoveSnap : _pfv3NudgeStep);
			Vector3 val4 = (((Object)(object)PFV3_Cam != (Object)null) ? ((Component)PFV3_Cam).transform.forward : Vector3.forward);
			_pfv3PreviewOffset += ((Vector3)(ref val4)).normalized * num11;
			PFV3_ApplyPreviewLocally();
			current.Use();
		}
		if (PFV3_LeftPressedThisFrame() && PFV3_RaycastNP(val, out var np2, out bestHit))
		{
			Keyboard current2 = Keyboard.current;
			int num12;
			if (current2 == null || !((ButtonControl)current2.leftCtrlKey).isPressed)
			{
				Keyboard current3 = Keyboard.current;
				num12 = ((current3 != null && ((ButtonControl)current3.rightCtrlKey).isPressed) ? 1 : 0);
			}
			else
			{
				num12 = 1;
			}
			string value2;
			string text = (_pfv3Hash2Name.TryGetValue(np2.Hash, out value2) ? value2 : ((Object)((Component)np2).gameObject).name);
			string text2 = PFV3_LeadingDigits(text) ?? np2.Hash.ToString();
			Transform transform = ((Component)np2).transform;
			PFV3Captured item = new PFV3Captured
			{
				Display = text,
				Id = text2,
				Hash = np2.Hash,
				Pos = transform.position,
				Rot = transform.rotation,
				Scale = transform.localScale,
				Tr = transform
			};
			if (num12 == 0)
			{
				_pfv3Sel.Items.Clear();
			}
			int num13 = _pfv3Sel.Items.FindIndex(delegate(PFV3Captured x)
			{
				//IL_0014: Unknown result type (might be due to invalid IL or missing references)
				//IL_001f: Unknown result type (might be due to invalid IL or missing references)
				//IL_0024: Unknown result type (might be due to invalid IL or missing references)
				//IL_0029: Unknown result type (might be due to invalid IL or missing references)
				if (x.Hash == item.Hash)
				{
					Vector3 val7 = x.Pos - item.Pos;
					return ((Vector3)(ref val7)).sqrMagnitude < 1E-05f;
				}
				return false;
			});
			if (num12 != 0 && num13 >= 0)
			{
				_pfv3Sel.Items.RemoveAt(num13);
			}
			else if (num13 < 0)
			{
				_pfv3Sel.Items.Add(item);
			}
			_pfv3Pivot = PFV3_ComputePivot();
			_pfv3Sel.Phase = ((_pfv3Sel.Items.Count > 0) ? PFV3SelPhase.Selected : PFV3SelPhase.Idle);
			SendConsoleCmd("select " + text2);
			_pfv3SelFocus = ((_pfv3Sel.Items.Count > 0) ? Mathf.Clamp((_pfv3SelFocus >= 0) ? _pfv3SelFocus : 0, 0, _pfv3Sel.Items.Count - 1) : (-1));
		}
		if (!_pfv3Dragging && _pfv3Sel.Phase >= PFV3SelPhase.Selected)
		{
			_pfv3HoveredAxis = PFV3_PickAxisHandle(_pfv3Pivot);
		}
		Keyboard current4 = Keyboard.current;
		if (current4 == null)
		{
			goto IL_084d;
		}
		int num14;
		if (!((ButtonControl)current4.leftCtrlKey).isPressed)
		{
			num14 = (((ButtonControl)current4.rightCtrlKey).isPressed ? 1 : 0);
			if (num14 == 0)
			{
				goto IL_0930;
			}
		}
		else
		{
			num14 = 1;
		}
		if (((ButtonControl)current4.zKey).wasPressedThisFrame)
		{
			PFV3_Undo();
		}
		goto IL_0930;
		IL_0930:
		if (num14 != 0 && ((ButtonControl)current4.yKey).wasPressedThisFrame)
		{
			PFV3_Redo();
		}
		goto IL_084d;
		IL_084d:
		if (_pfv3ShowRuler && PFV3_LeftPressedThisFrame())
		{
			Vector2 val5 = PFV3_MouseScreen();
			if (PFV3_Raycast(val5, out var hit))
			{
				_pfv3RulerPts.Add(((RaycastHit)(ref hit)).point);
			}
			else
			{
				Camera pFV3_Cam = PFV3_Cam;
				if ((Object)(object)pFV3_Cam != (Object)null)
				{
					Ray val6 = pFV3_Cam.ScreenPointToRay(new Vector3(val5.x, val5.y, 0f));
					if (Mathf.Abs(((Ray)(ref val6)).direction.y) > 1E-05f)
					{
						float num15 = (0f - ((Ray)(ref val6)).origin.y) / ((Ray)(ref val6)).direction.y;
						if (num15 > 0f)
						{
							_pfv3RulerPts.Add(((Ray)(ref val6)).origin + ((Ray)(ref val6)).direction * num15);
						}
					}
				}
			}
		}
		PFV3_BrushTick();
	}

	private static NetworkPrefab PFV3_FindNetworkPrefabRoot(Transform t)
	{
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Expected O, but got Unknown
		NetworkPrefab result = null;
		while ((Object)t != (Object)null)
		{
			if (((Component)t).TryGetComponent<NetworkPrefab>(ref result))
			{
				return result;
			}
			t = t.parent;
		}
		return null;
	}

	private bool PFV3_RaycastNP(Vector2 screen, out NetworkPrefab np, out RaycastHit bestHit)
	{
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_002e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0080: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b4: Expected O, but got Unknown
		//IL_00bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		Camera pFV3_Cam = PFV3_Cam;
		np = null;
		bestHit = default(RaycastHit);
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return false;
		}
		RaycastHit[] array = Physics.RaycastAll(pFV3_Cam.ScreenPointToRay(new Vector3(screen.x, screen.y, 0f)), 2000f, -1, (QueryTriggerInteraction)1);
		if (array == null || array.Length == 0)
		{
			return false;
		}
		Array.Sort(array, (RaycastHit a, RaycastHit b) => ((RaycastHit)(ref a)).distance.CompareTo(((RaycastHit)(ref b)).distance));
		RaycastHit[] array2 = array;
		for (int num = 0; num < array2.Length; num++)
		{
			RaycastHit val = array2[num];
			NetworkPrefab val2 = PFV3_FindNetworkPrefabRoot(((Object)(object)((RaycastHit)(ref val)).collider != (Object)null) ? ((Component)((RaycastHit)(ref val)).collider).transform : null);
			if ((Object)val2 != (Object)null)
			{
				np = val2;
				bestHit = val;
				return true;
			}
		}
		return false;
	}

	private List<PFV3TxtEntry> PFV3_ParseTxt(string path)
	{
		//IL_01e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_0245: Unknown result type (might be due to invalid IL or missing references)
		//IL_024a: Unknown result type (might be due to invalid IL or missing references)
		List<PFV3TxtEntry> list = new List<PFV3TxtEntry>();
		if (string.IsNullOrEmpty(path) || !File.Exists(path))
		{
			return list;
		}
		string text = File.ReadAllText(path).Trim();
		if (text.StartsWith("["))
		{
			text = text.Substring(1);
		}
		if (text.EndsWith("]"))
		{
			text = text.Substring(0, text.Length - 1);
		}
		string[] array = Regex.Split(text, "}\\s*,\\s*{");
		for (int i = 0; i < array.Length; i++)
		{
			string text2 = array[i];
			if (!text2.StartsWith("{"))
			{
				text2 = "{" + text2;
			}
			if (!text2.EndsWith("}"))
			{
				text2 += "}";
			}
			Match match = Regex.Match(text2, "'name'\\s*:\\s*'([^']+)'");
			Match match2 = Regex.Match(text2, "'hash'\\s*:\\s*(\\d+)");
			Match match3 = Regex.Match(text2, "'scale'\\s*:\\s*([-\\d\\.]+)");
			Match match4 = Regex.Match(text2, "'(?:pos|position)'\\s*:\\s*[\\(\\[]\\s*([-\\d\\.]+)\\s*,\\s*([-\\d\\.]+)\\s*,\\s*([-\\d\\.]+)\\s*[\\)\\]]");
			Match match5 = Regex.Match(text2, "'rotation'\\s*:\\s*[\\(\\[]\\s*([-\\d\\.]+)\\s*,\\s*([-\\d\\.]+)\\s*,\\s*([-\\d\\.]+)\\s*[\\)\\]]");
			if (match2.Success && match4.Success && match5.Success)
			{
				list.Add(new PFV3TxtEntry
				{
					Name = (match.Success ? match.Groups[1].Value : null),
					Hash = uint.Parse(match2.Groups[1].Value),
					Scale = (match3.Success ? float.Parse(match3.Groups[1].Value, CultureInfo.InvariantCulture) : 1f),
					Pos = new Vector3(float.Parse(match4.Groups[1].Value, CultureInfo.InvariantCulture), float.Parse(match4.Groups[2].Value, CultureInfo.InvariantCulture), float.Parse(match4.Groups[3].Value, CultureInfo.InvariantCulture)),
					Euler = new Vector3(float.Parse(match5.Groups[1].Value, CultureInfo.InvariantCulture), float.Parse(match5.Groups[2].Value, CultureInfo.InvariantCulture), float.Parse(match5.Groups[3].Value, CultureInfo.InvariantCulture))
				});
			}
		}
		return list;
	}

	private void PFV3_RefreshTxtList()
	{
		try
		{
			string pFV3_BlueprintDir = PFV3_BlueprintDir;
			_pfv3TxtFiles = Directory.GetFiles(pFV3_BlueprintDir, "*.txt", SearchOption.TopDirectoryOnly);
			Array.Sort(_pfv3TxtFiles, (IComparer<string>?)StringComparer.OrdinalIgnoreCase);
			if (_pfv3TxtFiles.Length == 0)
			{
				_pfv3TxtIndex = -1;
				_statusLeft = "TXT: no files in " + pFV3_BlueprintDir;
			}
			else if (_pfv3TxtIndex < 0 || _pfv3TxtIndex >= _pfv3TxtFiles.Length)
			{
				_pfv3TxtIndex = 0;
			}
		}
		catch (Exception)
		{
			_pfv3TxtFiles = Array.Empty<string>();
			_pfv3TxtIndex = -1;
		}
	}

	private void PFV3_LoadTxtIntoPreview(string path)
	{
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0043: Unknown result type (might be due to invalid IL or missing references)
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Expected O, but got Unknown
		//IL_030b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0310: Unknown result type (might be due to invalid IL or missing references)
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_019e: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01af: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ee: Expected O, but got Unknown
		//IL_0299: Unknown result type (might be due to invalid IL or missing references)
		//IL_029e: Unknown result type (might be due to invalid IL or missing references)
		//IL_02a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_02b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ba: Unknown result type (might be due to invalid IL or missing references)
		if (string.IsNullOrEmpty(path) || !File.Exists(path))
		{
			_statusLeft = "No file selected or file missing.";
			return;
		}
		_pfv3Sel.Items.Clear();
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
		if (File.ReadAllText(path).Trim().StartsWith("["))
		{
			List<PFV3TxtEntry> list = PFV3_ParseTxt(path);
			if (list.Count == 0)
			{
				_statusLeft = "No entries in file.";
				return;
			}
			if (_pfv3TxtPreview == null)
			{
				_pfv3TxtPreview = new PFV3TxtPreviewState();
			}
			foreach (GameObject item in _pfv3TxtPreview.Spawned)
			{
				if ((Object)(object)item != (Object)null)
				{
					Object.DestroyImmediate((Object)item);
				}
			}
			_pfv3TxtPreview.Spawned.Clear();
			foreach (PFV3TxtEntry item2 in list)
			{
				GameObject val = null;
				NetworkPrefab[] array = Object.FindObjectsOfType<NetworkPrefab>();
				NetworkPrefab[] array2 = array;
				foreach (NetworkPrefab val2 in array2)
				{
					if ((Object)(object)val2 != (Object)null && val2.Hash == item2.Hash)
					{
						val = ((Component)val2).gameObject;
						break;
					}
				}
				if ((Object)(object)val != (Object)null)
				{
					GameObject val3 = Object.Instantiate<GameObject>(val);
					((Object)val3).name = $"{item2.Hash}-Preview";
					Transform transform = val3.transform;
					transform.position = item2.Pos;
					transform.rotation = Quaternion.Euler(item2.Euler);
					transform.localScale = Vector3.one * Mathf.Max(0.0001f, item2.Scale);
					array = val3.GetComponentsInChildren<NetworkPrefab>(true);
					for (int j = 0; j < array.Length; j++)
					{
						Object.DestroyImmediate((Object)array[j], false);
					}
					_pfv3TxtPreview.Spawned.Add(val3);
					List<PFV3Captured> items = _pfv3Sel.Items;
					PFV3Captured pFV3Captured = new PFV3Captured();
					string text = item2.Name;
					uint hash;
					if (text == null)
					{
						hash = item2.Hash;
						text = hash.ToString();
					}
					pFV3Captured.Name = text;
					string text2 = item2.Name;
					if (text2 == null)
					{
						hash = item2.Hash;
						text2 = hash.ToString();
					}
					pFV3Captured.Display = text2;
					hash = item2.Hash;
					pFV3Captured.Id = hash.ToString();
					pFV3Captured.Hash = item2.Hash;
					pFV3Captured.Pos = transform.position;
					pFV3Captured.Rot = transform.rotation;
					pFV3Captured.Scale = transform.localScale;
					pFV3Captured.Tr = transform;
					items.Add(pFV3Captured);
				}
			}
			if (_pfv3Sel.Items.Count == 0)
			{
				_statusLeft = "No spawnable entries after matching hashes.";
				return;
			}
			_pfv3Pivot = PFV3_ComputePivot();
			_pfv3Sel.Active = true;
			_pfv3Sel.Phase = PFV3SelPhase.Selected;
			_pfv3SpawnPreviewActive = true;
			_pfv3SpawnPreviewFile = path;
			_pfv3Mode = PFV3Mode.Translate;
			_statusLeft = "Preview loaded: " + Path.GetFileName(path) + " — click Spawn again to send to server.";
		}
		else
		{
			File.ReadAllLines(path);
		}
	}

	private void PFV3_BuildTxtPreview(string filePath)
	{
		if (!string.IsNullOrEmpty(filePath) && File.Exists(filePath))
		{
			string[] lines = File.ReadAllLines(filePath);
			PFV3_BuildTxtPreview(lines, Path.GetFileNameWithoutExtension(filePath));
			_pfv3TxtPreview.SourcePath = filePath;
		}
	}

	private void PFV3_ClearTxtPreview()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		if ((Object)_pfv3TxtPreviewRoot != (Object)null)
		{
			Object.Destroy((Object)((Component)_pfv3TxtPreviewRoot).gameObject);
			_pfv3TxtPreviewRoot = null;
		}
		_pfv3TxtPreviewActive = false;
		_pfv3Sel.Items.Clear();
		_pfv3Sel.Active = false;
		_pfv3Sel.Phase = PFV3SelPhase.Idle;
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
	}

	private void PFV3_BuildTxtPreview(string[] lines, string label = null)
	{
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Expected O, but got Unknown
		//IL_007d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0085: Unknown result type (might be due to invalid IL or missing references)
		//IL_0451: Unknown result type (might be due to invalid IL or missing references)
		//IL_0456: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_02cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_02db: Unknown result type (might be due to invalid IL or missing references)
		//IL_02f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_0304: Expected O, but got Unknown
		//IL_0376: Unknown result type (might be due to invalid IL or missing references)
		//IL_037b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0383: Unknown result type (might be due to invalid IL or missing references)
		//IL_0388: Unknown result type (might be due to invalid IL or missing references)
		//IL_0390: Unknown result type (might be due to invalid IL or missing references)
		//IL_0395: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3TxtPreview == null)
		{
			_pfv3TxtPreview = new PFV3TxtPreviewState();
		}
		foreach (GameObject item in _pfv3TxtPreview.Spawned)
		{
			if ((Object)(object)item != (Object)null)
			{
				Object.DestroyImmediate((Object)item);
			}
		}
		_pfv3TxtPreview.Spawned.Clear();
		_pfv3TxtPreview.Captured.Clear();
		Vector3 position = default(Vector3);
		Quaternion rotation = default(Quaternion);
		for (int i = 0; i < lines.Length; i++)
		{
			string text = lines[i]?.Trim();
			if (string.IsNullOrEmpty(text) || text.StartsWith("#"))
			{
				continue;
			}
			if (text.StartsWith("spawn", StringComparison.OrdinalIgnoreCase))
			{
				int num = text.IndexOf("string-raw", StringComparison.OrdinalIgnoreCase);
				if (num >= 0)
				{
					text = text.Substring(num + "string-raw".Length).Trim();
				}
			}
			if (text.StartsWith(","))
			{
				text = text.Substring(1);
			}
			string[] array = text.Split(new char[1] { ',' });
			if (array.Length < 11 || !uint.TryParse(array[0], out var result) || !float.TryParse(array[3], NumberStyles.Float, Inv, out var result2) || !float.TryParse(array[4], NumberStyles.Float, Inv, out var result3) || !float.TryParse(array[5], NumberStyles.Float, Inv, out var result4) || !float.TryParse(array[6], NumberStyles.Float, Inv, out var result5) || !float.TryParse(array[7], NumberStyles.Float, Inv, out var result6) || !float.TryParse(array[8], NumberStyles.Float, Inv, out var result7) || !float.TryParse(array[9], NumberStyles.Float, Inv, out var result8) || !float.TryParse(array[10], NumberStyles.Float, Inv, out var result9))
			{
				continue;
			}
			((Vector3)(ref position))._002Ector(result2, result3, result4);
			((Quaternion)(ref rotation))._002Ector(result5, result6, result7, result8);
			float num2 = Mathf.Max(0.0001f, result9);
			GameObject val = null;
			NetworkPrefab[] array2 = Object.FindObjectsOfType<NetworkPrefab>();
			NetworkPrefab[] array3 = array2;
			foreach (NetworkPrefab val2 in array3)
			{
				if ((Object)(object)val2 != (Object)null && val2.Hash == result)
				{
					val = ((Component)val2).gameObject;
					break;
				}
			}
			if ((Object)(object)val != (Object)null)
			{
				GameObject val3 = Object.Instantiate<GameObject>(val);
				((Object)val3).name = $"{result}-Preview";
				Transform transform = val3.transform;
				transform.position = position;
				transform.rotation = rotation;
				transform.localScale = Vector3.one * num2;
				array2 = val3.GetComponentsInChildren<NetworkPrefab>(true);
				for (int k = 0; k < array2.Length; k++)
				{
					Object.DestroyImmediate((Object)array2[k], false);
				}
				_pfv3TxtPreview.Spawned.Add(val3);
				_pfv3TxtPreview.Captured.Add(new PFV3Captured
				{
					Name = ((Object)val3).name,
					Display = ((Object)val3).name,
					Id = (PFV3_LeadingDigits(((Object)val3).name) ?? result.ToString()),
					Hash = result,
					Pos = transform.position,
					Rot = transform.rotation,
					Scale = transform.localScale,
					Tr = transform
				});
			}
		}
		_pfv3TxtPreview.Active = _pfv3TxtPreview.Spawned.Count > 0;
		_pfv3TxtPreview.Label = label ?? "Txt Preview";
		_pfv3Sel.Items.Clear();
		_pfv3Sel.Items.AddRange(_pfv3TxtPreview.Captured);
		_pfv3Sel.Active = _pfv3Sel.Items.Count > 0;
		_pfv3Sel.Phase = (_pfv3Sel.Active ? PFV3SelPhase.Selected : PFV3SelPhase.Idle);
		_pfv3Pivot = PFV3_ComputePivot();
		PFV3_ApplyPreviewLocally();
	}

	private void PFV3_SaveToText(bool nearby, string name)
	{
		//IL_00b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_014e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0145: Unknown result type (might be due to invalid IL or missing references)
		//IL_0153: Unknown result type (might be due to invalid IL or missing references)
		//IL_016a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0161: Unknown result type (might be due to invalid IL or missing references)
		//IL_016f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0186: Unknown result type (might be due to invalid IL or missing references)
		//IL_017d: Unknown result type (might be due to invalid IL or missing references)
		//IL_018b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0192: Unknown result type (might be due to invalid IL or missing references)
		//IL_0199: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bf: Unknown result type (might be due to invalid IL or missing references)
		string pFV3_BlueprintDir = PFV3_BlueprintDir;
		if (string.IsNullOrWhiteSpace(name))
		{
			name = DateTime.Now.ToString("yyyyMMdd_HHmmss");
		}
		string text = Path.GetFileNameWithoutExtension(name) + ".txt";
		try
		{
			Directory.CreateDirectory(pFV3_BlueprintDir);
		}
		catch
		{
		}
		string outPath = Path.Combine(pFV3_BlueprintDir, text);
		List<string> list = new List<string>();
		if (nearby)
		{
			foreach (PFV3Captured item3 in _pfv3Scan)
			{
				float uniformScale = Mathf.Max(0.0001f, (item3.Scale.x + item3.Scale.y + item3.Scale.z) / 3f);
				string item = PFV3_BuildSpawnCsv(item3.Hash, item3.Pos, item3.Rot, uniformScale, _pfv3Kinematic);
				list.Add(item);
			}
		}
		else
		{
			foreach (PFV3Captured item4 in _pfv3Sel.Items)
			{
				Transform val = (((Object)(object)item4.Tr != (Object)null) ? item4.Tr : PFV3_FindTransformFor(item4));
				Vector3 pos = (((Object)(object)val != (Object)null) ? val.position : item4.Pos);
				Quaternion rot = (((Object)(object)val != (Object)null) ? val.rotation : item4.Rot);
				Vector3 val2 = (((Object)(object)val != (Object)null) ? val.localScale : item4.Scale);
				float uniformScale2 = Mathf.Max(0.0001f, (val2.x + val2.y + val2.z) / 3f);
				string item2 = PFV3_BuildSpawnCsv(item4.Hash, pos, rot, uniformScale2, _pfv3Kinematic);
				list.Add(item2);
			}
		}
		try
		{
			File.WriteAllLines(outPath, list);
			PFV3_RefreshTxtList();
			_pfv3TxtIndex = Array.FindIndex(_pfv3TxtFiles, (string p) => string.Equals(p, outPath, StringComparison.OrdinalIgnoreCase));
			_statusLeft = $"Saved {list.Count} to '{text}'.";
		}
		catch (Exception arg)
		{
			MelonLogger.Error($"PFV3_SaveToText: {arg}");
			_statusLeft = "Save failed.";
		}
	}

	private void PFV3_DoScan()
	{
		//IL_0051: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_018f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0194: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a6: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_010b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0110: Unknown result type (might be due to invalid IL or missing references)
		//IL_011d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0122: Unknown result type (might be due to invalid IL or missing references)
		_pfv3Scan.Clear();
		Transform val = Actions.find_head();
		if ((Object)(object)val == (Object)null)
		{
			return;
		}
		Regex regex = new Regex("^(\\d+)\\s*-\\s*(.+)$");
		GameObject[] array = Object.FindObjectsOfType<GameObject>();
		NetworkPrefab val2 = null;
		GameObject[] array2 = array;
		foreach (GameObject val3 in array2)
		{
			if (val3.TryGetComponent<NetworkPrefab>(ref val2) && !(Vector3.Distance(val3.transform.position, val.position) > _scanRadius))
			{
				Match match = regex.Match(((Object)val3).name);
				if (match.Success)
				{
					string value = match.Groups[1].Value;
					string value2 = match.Groups[2].Value;
					_pfv3Scan.Add(new PFV3Captured
					{
						Display = value + " - " + value2,
						Id = value,
						Hash = (uint.TryParse(value, out var result) ? result : val2.Hash),
						Pos = val3.transform.position,
						Rot = val3.transform.rotation,
						Scale = val3.transform.localScale
					});
				}
				else
				{
					PFV3_LeadingDigits(((Object)val3).name);
					_pfv3Scan.Add(new PFV3Captured
					{
						Display = ((Object)val3).name,
						Id = (PFV3_LeadingDigits(((Object)val3).name) ?? val2.Hash.ToString()),
						Hash = val2.Hash,
						Pos = val3.transform.position,
						Rot = val3.transform.rotation,
						Scale = val3.transform.localScale
					});
				}
			}
		}
		_statusLeft = $"Scanner: {_pfv3Scan.Count} prefabs within {_scanRadius:0.0}m.";
	}

	private void PFV3_BrushTick()
	{
		//IL_006f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
		if (!_pfv3EditorOn)
		{
			if ((Object)(object)brushSphere != (Object)null)
			{
				brushSphere.SetActive(false);
			}
			return;
		}
		if (!brushModeActive)
		{
			if ((Object)(object)brushSphere != (Object)null)
			{
				brushSphere.SetActive(false);
			}
			return;
		}
		if ((Object)(object)brushSphere != (Object)null)
		{
			brushSphere.SetActive(true);
		}
		TryPlaceBrush();
		CollectItemsInBrush();
		Mouse current = Mouse.current;
		if (current == null)
		{
			return;
		}
		Vector2 mouseBL = PFV3_MouseScreen();
		bool num = PFV3_Block3DInput(mouseBL);
		if (!num)
		{
			float y = ((InputControl<Vector2>)(object)current.scroll).ReadValue().y;
			if (Mathf.Abs(y) > 0.01f)
			{
				brushSize = Mathf.Clamp(brushSize + Mathf.Sign(y) * 0.3f, 0.5f, 20f);
				if ((Object)(object)brushSphere != (Object)null)
				{
					brushSphere.transform.localScale = Vector3.one * brushSize;
				}
			}
		}
		if (!num && current.leftButton.wasPressedThisFrame && !isRunningPickupRoutine && collectedItems.Count > 0)
		{
			isRunningPickupRoutine = true;
			MelonCoroutines.Start(RunPickupRoutine());
		}
		if (!num && current.rightButton.wasPressedThisFrame)
		{
			EraseObjectsInBrush();
		}
	}

	private void CollectItemsInBrush()
	{
		//IL_0005: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Expected O, but got Unknown
		if ((Object)brushSphere == (Object)null)
		{
			return;
		}
		Vector3 position = brushSphere.transform.position;
		float num = 0.5f * brushSize;
		Collider[] array = Physics.OverlapSphere(position, num);
		for (int i = 0; i < array.Length; i++)
		{
			Pickup componentInParent = ((Component)array[i]).gameObject.GetComponentInParent<Pickup>();
			if ((Object)componentInParent != (Object)null)
			{
				GameObject gameObject = ((Component)componentInParent).gameObject;
				if (!collectedItems.Contains(gameObject))
				{
					collectedItems.Add(gameObject);
					HighlightObject(gameObject);
					MelonLogger.Msg("Collected Pickup: " + ((Object)gameObject).name);
				}
			}
		}
	}

	private void CreateHighlightMaterial()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Expected O, but got Unknown
		//IL_002e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_005d: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)highlightMaterial == (Object)null)
		{
			highlightMaterial = new Material(Shader.Find("Standard"));
			highlightMaterial.color = Color.yellow;
			highlightMaterial.EnableKeyword("_EMISSION");
			highlightMaterial.SetColor("_EmissionColor", Color.yellow * 0.8f);
		}
	}

	private void HighlightObject(GameObject obj)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if ((Object)obj == (Object)null || originalMaterials.ContainsKey(obj))
		{
			return;
		}
		Renderer[] componentsInChildren = obj.GetComponentsInChildren<Renderer>();
		if (componentsInChildren.Length != 0)
		{
			CreateHighlightMaterial();
			Material[] array = (Material[])(object)new Material[componentsInChildren.Length];
			for (int i = 0; i < componentsInChildren.Length; i++)
			{
				array[i] = componentsInChildren[i].material;
				componentsInChildren[i].material = highlightMaterial;
			}
			originalMaterials[obj] = array;
		}
	}

	private void RemoveHighlight(GameObject obj)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Expected O, but got Unknown
		if (!((Object)obj == (Object)null) && originalMaterials.ContainsKey(obj))
		{
			Renderer[] componentsInChildren = obj.GetComponentsInChildren<Renderer>();
			Material[] array = originalMaterials[obj];
			int num = Mathf.Min(componentsInChildren.Length, array.Length);
			for (int i = 0; i < num; i++)
			{
				componentsInChildren[i].material = array[i];
			}
			originalMaterials.Remove(obj);
		}
	}

	private void ClearAllHighlights()
	{
		foreach (GameObject collectedItem in collectedItems)
		{
			RemoveHighlight(collectedItem);
		}
		collectedItems.Clear();
		originalMaterials.Clear();
	}

	private void EraseObjectsInBrush()
	{
		//IL_0005: Unknown result type (might be due to invalid IL or missing references)
		//IL_0010: Expected O, but got Unknown
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_005a: Expected O, but got Unknown
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)brushSphere == (Object)null)
		{
			return;
		}
		Vector3 position = brushSphere.transform.position;
		float num = 0.5f * brushSize;
		List<GameObject> list = new List<GameObject>();
		foreach (GameObject collectedItem in collectedItems)
		{
			if ((Object)collectedItem == (Object)null || Vector3.Distance(collectedItem.transform.position, position) <= num)
			{
				list.Add(collectedItem);
			}
		}
		foreach (GameObject item in list)
		{
			RemoveHighlight(item);
			collectedItems.Remove(item);
			MelonLogger.Msg("Erased Pickup: " + (((Object)(object)item != (Object)null) ? ((Object)item).name : "(null)"));
		}
	}

	private IEnumerator PickupAndDrop(Pickup pickup, Interactor interactor)
	{
		if (!((Object)pickup == (Object)null) && !((Object)interactor == (Object)null))
		{
			interactor.ResetTimeout();
			interactor.StartInteract((Interactable)pickup, false, false, false);
			yield return (object)new WaitForSeconds(0.05f);
			if (interactor.IsInteracting)
			{
				interactor.StopInteract(false, false);
			}
		}
	}

	private IEnumerator PickupAllCollectedSequentially()
	{
		PlayerController current = PlayerController.Current;
		object obj;
		if ((Object)(object)current == (Object)null)
		{
			obj = null;
		}
		else
		{
			Controller rightController = current.RightController;
			obj = (((Object)(object)rightController != (Object)null) ? rightController.Interactor : null);
		}
		Interactor interactor = (Interactor)obj;
		if ((Object)interactor == (Object)null)
		{
			yield break;
		}
		List<GameObject> list = new List<GameObject>(collectedItems);
		foreach (GameObject item in list)
		{
			if ((Object)(object)item != (Object)null)
			{
				Pickup componentInParent = item.GetComponentInParent<Pickup>();
				if ((Object)componentInParent != (Object)null)
				{
					yield return PickupAndDrop(componentInParent, interactor);
				}
			}
		}
	}

	private IEnumerator RunPickupRoutine()
	{
		yield return PickupAllCollectedSequentially();
		isRunningPickupRoutine = false;
	}

	private void TryPlaceBrush()
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Expected O, but got Unknown
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_0046: Unknown result type (might be due to invalid IL or missing references)
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_0093: Expected O, but got Unknown
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_0095: Unknown result type (might be due to invalid IL or missing references)
		if (!_pfv3EditorOn)
		{
			return;
		}
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)pFV3_Cam == (Object)null)
		{
			return;
		}
		Vector2 val = PFV3_MouseScreen();
		if (PFV3_Block3DInput(val))
		{
			return;
		}
		Ray val2 = pFV3_Cam.ScreenPointToRay(new Vector3(val.x, val.y, 0f));
		int num = ~LayerMask.GetMask(new string[1] { "Player" });
		RaycastHit val3 = default(RaycastHit);
		if (Physics.Raycast(val2, ref val3, 1000f, num, (QueryTriggerInteraction)1))
		{
			Vector3 point = ((RaycastHit)(ref val3)).point;
			if ((Object)brushSphere == (Object)null)
			{
				brushSphere = CreateWireframeSphere(point, brushSize);
				((Object)brushSphere).name = "PFV3_BrushSphere";
			}
			else
			{
				brushSphere.transform.position = point;
				brushSphere.transform.localScale = Vector3.one * brushSize;
			}
		}
	}

	public static GameObject CreateWireframeSphere(Vector3 center, float diameter, int segments = 64)
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Expected O, but got Unknown
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		GameObject root = new GameObject("WireSphere");
		root.transform.position = center;
		root.transform.localScale = Vector3.one * Mathf.Max(0.01f, diameter);
		MakeCircle("Ring_X", Vector3.right);
		MakeCircle("Ring_Y", Vector3.up);
		MakeCircle("Ring_Z", Vector3.forward);
		return root;
		void MakeCircle(string name, Vector3 axis)
		{
			//IL_0001: Unknown result type (might be due to invalid IL or missing references)
			//IL_0006: Unknown result type (might be due to invalid IL or missing references)
			//IL_0064: Unknown result type (might be due to invalid IL or missing references)
			//IL_0069: Unknown result type (might be due to invalid IL or missing references)
			//IL_007e: Unknown result type (might be due to invalid IL or missing references)
			//IL_008d: Expected O, but got Unknown
			//IL_008f: Unknown result type (might be due to invalid IL or missing references)
			//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
			//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
			//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
			//IL_00de: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e3: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e6: Unknown result type (might be due to invalid IL or missing references)
			//IL_00f0: Unknown result type (might be due to invalid IL or missing references)
			GameObject val = new GameObject(name);
			val.transform.SetParent(root.transform, false);
			LineRenderer val2 = val.AddComponent<LineRenderer>();
			val2.useWorldSpace = false;
			val2.loop = true;
			val2.positionCount = Mathf.Max(8, segments);
			float num = (val2.endWidth = 0.01f);
			float startWidth = num;
			val2.startWidth = startWidth;
			((Renderer)val2).material = new Material(Shader.Find("Unlit/Color"))
			{
				color = new Color(1f, 0.85f, 0.1f, 0.9f)
			};
			Vector3 val3 = default(Vector3);
			for (int i = 0; i < val2.positionCount; i++)
			{
				float num3 = (float)i / (float)val2.positionCount * (float)Math.PI * 2f;
				((Vector3)(ref val3))._002Ector(Mathf.Cos(num3), 0f, Mathf.Sin(num3));
				Quaternion val4 = Quaternion.FromToRotation(Vector3.up, ((Vector3)(ref axis)).normalized);
				val2.SetPosition(i, val4 * val3 * 0.5f);
			}
		}
	}

	private static Vector2 PFV3_MouseScreen()
	{
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		Mouse current = Mouse.current;
		if (current == null)
		{
			return new Vector2((float)Screen.width * 0.5f, (float)Screen.height * 0.5f);
		}
		return ((InputControl<Vector2>)(object)((Pointer)current).position).ReadValue();
	}

	private static bool PFV3_LeftPressedThisFrame()
	{
		Mouse current = Mouse.current;
		if (current != null)
		{
			return current.leftButton.wasPressedThisFrame;
		}
		return false;
	}

	private void PFV3_HandleInput()
	{
		//IL_000d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_0034: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_009d: Invalid comparison between Unknown and I4
		//IL_0188: Unknown result type (might be due to invalid IL or missing references)
		//IL_028e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0294: Invalid comparison between Unknown and I4
		//IL_05bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_05c3: Invalid comparison between Unknown and I4
		//IL_01b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_0692: Unknown result type (might be due to invalid IL or missing references)
		//IL_0698: Invalid comparison between Unknown and I4
		//IL_0178: Unknown result type (might be due to invalid IL or missing references)
		//IL_0119: Unknown result type (might be due to invalid IL or missing references)
		//IL_0124: Expected O, but got Unknown
		//IL_0624: Unknown result type (might be due to invalid IL or missing references)
		//IL_0629: Unknown result type (might be due to invalid IL or missing references)
		//IL_064c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0651: Unknown result type (might be due to invalid IL or missing references)
		//IL_0656: Unknown result type (might be due to invalid IL or missing references)
		//IL_0658: Unknown result type (might be due to invalid IL or missing references)
		//IL_065a: Unknown result type (might be due to invalid IL or missing references)
		//IL_065e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0663: Unknown result type (might be due to invalid IL or missing references)
		//IL_0665: Unknown result type (might be due to invalid IL or missing references)
		//IL_066a: Unknown result type (might be due to invalid IL or missing references)
		//IL_066f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0675: Unknown result type (might be due to invalid IL or missing references)
		//IL_067a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c98: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c9d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a99: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a9f: Invalid comparison between Unknown and I4
		//IL_01e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ee: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f3: Unknown result type (might be due to invalid IL or missing references)
		//IL_0204: Unknown result type (might be due to invalid IL or missing references)
		//IL_0209: Unknown result type (might be due to invalid IL or missing references)
		//IL_0210: Unknown result type (might be due to invalid IL or missing references)
		//IL_0215: Unknown result type (might be due to invalid IL or missing references)
		//IL_021d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a13: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a6d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a7e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0a83: Unknown result type (might be due to invalid IL or missing references)
		//IL_0242: Unknown result type (might be due to invalid IL or missing references)
		//IL_0239: Unknown result type (might be due to invalid IL or missing references)
		//IL_0506: Unknown result type (might be due to invalid IL or missing references)
		//IL_050c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0512: Unknown result type (might be due to invalid IL or missing references)
		//IL_040d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0412: Unknown result type (might be due to invalid IL or missing references)
		//IL_0416: Unknown result type (might be due to invalid IL or missing references)
		//IL_041c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0421: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_02dc: Unknown result type (might be due to invalid IL or missing references)
		//IL_02df: Unknown result type (might be due to invalid IL or missing references)
		//IL_02e4: Unknown result type (might be due to invalid IL or missing references)
		//IL_02e7: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_02fc: Unknown result type (might be due to invalid IL or missing references)
		//IL_02fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_0303: Unknown result type (might be due to invalid IL or missing references)
		//IL_0305: Unknown result type (might be due to invalid IL or missing references)
		//IL_030a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0247: Unknown result type (might be due to invalid IL or missing references)
		//IL_024e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0253: Unknown result type (might be due to invalid IL or missing references)
		//IL_025a: Unknown result type (might be due to invalid IL or missing references)
		//IL_025f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0523: Unknown result type (might be due to invalid IL or missing references)
		//IL_0526: Unknown result type (might be due to invalid IL or missing references)
		//IL_052b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0530: Unknown result type (might be due to invalid IL or missing references)
		//IL_0597: Unknown result type (might be due to invalid IL or missing references)
		//IL_059e: Unknown result type (might be due to invalid IL or missing references)
		//IL_05a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_0431: Unknown result type (might be due to invalid IL or missing references)
		//IL_0437: Unknown result type (might be due to invalid IL or missing references)
		//IL_043c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0376: Unknown result type (might be due to invalid IL or missing references)
		//IL_0364: Unknown result type (might be due to invalid IL or missing references)
		//IL_0339: Unknown result type (might be due to invalid IL or missing references)
		//IL_033e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0342: Unknown result type (might be due to invalid IL or missing references)
		//IL_0347: Unknown result type (might be due to invalid IL or missing references)
		//IL_034c: Unknown result type (might be due to invalid IL or missing references)
		//IL_06ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_044a: Unknown result type (might be due to invalid IL or missing references)
		//IL_044d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0452: Unknown result type (might be due to invalid IL or missing references)
		//IL_0457: Unknown result type (might be due to invalid IL or missing references)
		//IL_045b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0460: Unknown result type (might be due to invalid IL or missing references)
		//IL_0463: Unknown result type (might be due to invalid IL or missing references)
		//IL_0468: Unknown result type (might be due to invalid IL or missing references)
		//IL_046d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0471: Unknown result type (might be due to invalid IL or missing references)
		//IL_0476: Unknown result type (might be due to invalid IL or missing references)
		//IL_0478: Unknown result type (might be due to invalid IL or missing references)
		//IL_047a: Unknown result type (might be due to invalid IL or missing references)
		//IL_04a4: Unknown result type (might be due to invalid IL or missing references)
		//IL_04a9: Unknown result type (might be due to invalid IL or missing references)
		//IL_037b: Unknown result type (might be due to invalid IL or missing references)
		//IL_037e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0384: Unknown result type (might be due to invalid IL or missing references)
		//IL_0389: Unknown result type (might be due to invalid IL or missing references)
		//IL_0bf7: Unknown result type (might be due to invalid IL or missing references)
		//IL_0bfc: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c02: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c04: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c11: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c16: Unknown result type (might be due to invalid IL or missing references)
		//IL_0765: Unknown result type (might be due to invalid IL or missing references)
		//IL_0771: Unknown result type (might be due to invalid IL or missing references)
		//IL_0777: Unknown result type (might be due to invalid IL or missing references)
		//IL_070c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0712: Unknown result type (might be due to invalid IL or missing references)
		//IL_0717: Unknown result type (might be due to invalid IL or missing references)
		//IL_071c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c2f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0c34: Unknown result type (might be due to invalid IL or missing references)
		//IL_0af1: Unknown result type (might be due to invalid IL or missing references)
		//IL_0afc: Expected O, but got Unknown
		//IL_07d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_07a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_07ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_07b6: Unknown result type (might be due to invalid IL or missing references)
		//IL_07bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_07c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_0739: Unknown result type (might be due to invalid IL or missing references)
		//IL_073e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0746: Unknown result type (might be due to invalid IL or missing references)
		//IL_074b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0750: Unknown result type (might be due to invalid IL or missing references)
		//IL_0722: Unknown result type (might be due to invalid IL or missing references)
		//IL_0730: Unknown result type (might be due to invalid IL or missing references)
		//IL_0735: Unknown result type (might be due to invalid IL or missing references)
		//IL_03c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_03c9: Unknown result type (might be due to invalid IL or missing references)
		//IL_03cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_03d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_03d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b07: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b0c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b1a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b1f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b2d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b32: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b51: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b56: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b5c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b61: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b67: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b6c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b73: Unknown result type (might be due to invalid IL or missing references)
		//IL_0b78: Unknown result type (might be due to invalid IL or missing references)
		//IL_0908: Unknown result type (might be due to invalid IL or missing references)
		//IL_083e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0843: Unknown result type (might be due to invalid IL or missing references)
		//IL_04d6: Unknown result type (might be due to invalid IL or missing references)
		//IL_04db: Unknown result type (might be due to invalid IL or missing references)
		//IL_04dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_04e2: Unknown result type (might be due to invalid IL or missing references)
		//IL_087a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0868: Unknown result type (might be due to invalid IL or missing references)
		//IL_0945: Unknown result type (might be due to invalid IL or missing references)
		//IL_087f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0886: Unknown result type (might be due to invalid IL or missing references)
		//IL_088c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0892: Unknown result type (might be due to invalid IL or missing references)
		//IL_094a: Unknown result type (might be due to invalid IL or missing references)
		//IL_094e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0954: Unknown result type (might be due to invalid IL or missing references)
		//IL_0959: Unknown result type (might be due to invalid IL or missing references)
		//IL_0960: Unknown result type (might be due to invalid IL or missing references)
		//IL_093e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0937: Unknown result type (might be due to invalid IL or missing references)
		//IL_08ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_08d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_08d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_08dc: Unknown result type (might be due to invalid IL or missing references)
		//IL_08a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_08a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_08a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_08ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_09ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_09d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_08f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_08fc: Unknown result type (might be due to invalid IL or missing references)
		_pfv3TabFocus = true;
		Vector2 val = PFV3_MouseScreen();
		if (PFV3_Block3DInput(val))
		{
			return;
		}
		PFV3_Cam.ScreenPointToRay(new Vector3(val.x, val.y, 0f));
		bool num = val.y <= 68f;
		bool flag = val.x >= (float)Screen.width - 360f && val.y >= 68f;
		if (num || flag || !_pfv3EditorOn || !_pfv3TabFocus)
		{
			return;
		}
		Event current = Event.current;
		if (current == null)
		{
			return;
		}
		if ((int)current.type != 1 || current.button != 0 || !_pfv3Sel.Active || _pfv3Sel.Phase != PFV3SelPhase.DraggingRect)
		{
			goto IL_0187;
		}
		Vector2 val2 = current.mousePosition - _pfv3MouseStart;
		float magnitude = ((Vector2)(ref val2)).magnitude;
		_pfv3Sel.Items.Clear();
		GameObject val3;
		Vector3 pos;
		Quaternion rot;
		uint hash;
		string name;
		int num2;
		if (magnitude < 6f)
		{
			val3 = PFV3_RayPickPrefab(out pos, out rot, out hash, out name);
			if ((Object)val3 != (Object)null)
			{
				if (Keyboard.current != null)
				{
					if (!((ButtonControl)Keyboard.current.leftCtrlKey).isPressed)
					{
						num2 = (((ButtonControl)Keyboard.current.rightCtrlKey).isPressed ? 1 : 0);
						if (num2 == 0)
						{
							goto IL_0ced;
						}
					}
					else
					{
						num2 = 1;
					}
					goto IL_0b7e;
				}
				num2 = 0;
				goto IL_0ced;
			}
		}
		else
		{
			PFV3_CaptureInBounds(_pfv3Sel.Bounds);
		}
		goto IL_0c96;
		IL_0187:
		Vector3 val4;
		if ((int)current.type == 0 && current.button == 0 && _pfv3Sel.Phase == PFV3SelPhase.Selected)
		{
			int num3 = PFV3_PickAxisHandle(_pfv3Pivot);
			if (num3 >= 0)
			{
				_pfv3ActiveAxis = ((_pfv3Mode == PFV3Mode.Scale) ? 3 : num3);
				_pfv3Dragging = true;
				Bounds bounds = _pfv3Sel.Bounds;
				val4 = ((Bounds)(ref bounds)).extents;
				_pfv3ScaleRefLen = ((Vector3)(ref val4)).magnitude;
				_pfv3StartScale = _pfv3PreviewScale;
				_pfv3MouseStart = current.mousePosition;
				_pfv3DragStartWorld = (Vector3)(((_003F?)PFV3_ClosestPointOnAxis(_pfv3Pivot, _pfv3ActiveAxis)) ?? _pfv3Pivot);
				_pfv3DragStartOffset = _pfv3PreviewOffset;
				_pfv3StartEulerAdd = _pfv3PreviewEuler;
				_pfv3Sel.Phase = PFV3SelPhase.Preview;
				_pfv3Sel.Active = true;
				current.Use();
			}
		}
		if (_pfv3Dragging && (int)current.type == 3 && current.button == 0)
		{
			Vector3 world3;
			if (_pfv3Mode == PFV3Mode.Translate)
			{
				int num4 = ((_pfv3ActiveAxis >= 0 && _pfv3ActiveAxis <= 2) ? _pfv3ActiveAxis : (-1));
				if (num4 >= 0)
				{
					Vector3 val5 = PFV3_GetAxisDir(num4);
					Vector3 pfv3Pivot = _pfv3Pivot;
					Vector3 pfv3DragStartWorld = _pfv3DragStartWorld;
					float num5 = Vector3.Dot((PFV3_ClosestPointOnAxis(pfv3Pivot, num4) ?? pfv3DragStartWorld) - pfv3DragStartWorld, val5);
					if (_pfv3MoveSnap > 0.0001f)
					{
						num5 = Mathf.Round(num5 / _pfv3MoveSnap) * _pfv3MoveSnap;
					}
					_pfv3PreviewOffset = _pfv3DragStartOffset + val5 * num5;
				}
				else
				{
					Vector3 val6 = (((Object)(object)PFV3_Cam != (Object)null) ? ((Component)PFV3_Cam).transform.forward : Vector3.forward);
					float num6 = (current.mousePosition - _pfv3MouseStart).y * 0.01f;
					float num7 = ((_pfv3MoveSnap > 0.0001f) ? _pfv3MoveSnap : 0.01f);
					float num8 = Mathf.Round(num6 / num7) * num7;
					_pfv3PreviewOffset = _pfv3DragStartOffset + val6 * num8;
				}
			}
			else if (_pfv3Mode == PFV3Mode.Rotate)
			{
				int num9 = ((_pfv3ActiveAxis < 0 || _pfv3ActiveAxis > 2) ? 1 : _pfv3ActiveAxis);
				Vector3 val7 = PFV3_GetAxisDir(num9);
				if (PFV3_RaycastToPlane(current.mousePosition, _pfv3Pivot, val7, out var world) && PFV3_RaycastToPlane(_pfv3MouseStart, _pfv3Pivot, val7, out var world2))
				{
					val4 = world2 - _pfv3Pivot;
					Vector3 normalized = ((Vector3)(ref val4)).normalized;
					val4 = world - _pfv3Pivot;
					Vector3 normalized2 = ((Vector3)(ref val4)).normalized;
					float num10 = Vector3.SignedAngle(normalized, normalized2, val7);
					float num11 = Mathf.Max(1f, _pfv3RotateSnap);
					num10 = Mathf.Round(num10 / num11) * num11;
					Vector3 zero = Vector3.zero;
					if (num9 == 0)
					{
						zero.x = num10;
					}
					if (num9 == 1)
					{
						zero.y = num10;
					}
					if (num9 == 2)
					{
						zero.z = num10;
					}
					_pfv3PreviewEuler = _pfv3StartEulerAdd + zero;
				}
			}
			else if (_pfv3Mode == PFV3Mode.Scale && _pfv3ActiveAxis == 3 && PFV3_RaycastToPlane(current.mousePosition, _pfv3Pivot, _pfv3DragPlaneNormal, out world3))
			{
				val4 = world3 - _pfv3Pivot;
				float num12 = Mathf.Clamp(Mathf.Max(0.001f, ((Vector3)(ref val4)).magnitude) / _pfv3ScaleStartRadius, 0.001f, 1000f);
				float num13 = _pfv3StartScale.x * num12;
				float num14 = Mathf.Max(0.001f, _pfv3ScaleSnap);
				float num15 = Mathf.Max(0.001f, Mathf.Round(num13 / num14) * num14);
				_pfv3PreviewScale = Vector3.one * num15;
			}
			PFV3_ApplyPreviewLocally();
			current.Use();
		}
		if (_pfv3Dragging && (int)current.type == 1 && current.button == 0)
		{
			_pfv3Dragging = false;
			_pfv3ActiveAxis = -1;
			_pfv3Sel.Phase = PFV3SelPhase.Selected;
			current.Use();
		}
		if (Time.time >= _pfv3NextSendAt)
		{
			if (_pfv3Sel.Items.Count > 0)
			{
				PFV3Captured pFV3Captured = _pfv3Sel.Items[0];
				Vector3 pfv3Pivot2 = _pfv3Pivot;
				Quaternion val8 = Quaternion.Euler(new Vector3(_pfv3PreviewEuler.y, _pfv3PreviewEuler.x, _pfv3PreviewEuler.z));
				_ = pfv3Pivot2 + val8 * (pFV3Captured.Pos - pfv3Pivot2) + _pfv3PreviewOffset;
			}
			_pfv3NextSendAt = Time.time + 0.1f;
		}
		if ((int)current.type == 3 && current.button == 0 && _pfv3Dragging)
		{
			bool flag2 = Keyboard.current != null && (((ButtonControl)Keyboard.current.leftCtrlKey).isPressed || ((ButtonControl)Keyboard.current.rightCtrlKey).isPressed);
			if (_pfv3Mode == PFV3Mode.Translate)
			{
				Vector3? val9 = PFV3_ClosestPointOnAxis(_pfv3Pivot, _pfv3ActiveAxis);
				if (val9.HasValue)
				{
					Vector3 val10 = val9.Value - _pfv3DragStartWorld;
					if (flag2)
					{
						val10 = PFV3_SnapAxis(val10, _pfv3ActiveAxis, _pfv3MoveSnap);
					}
					_pfv3PreviewOffset = _pfv3DragStartOffset + PFV3_AxisMask(val10, _pfv3ActiveAxis);
				}
			}
			else if (_pfv3Mode == PFV3Mode.Rotate)
			{
				float num16 = PFV3_AngleDeltaOnRing(_pfv3Pivot, _pfv3ActiveAxis, _pfv3MouseStart, current.mousePosition);
				if (flag2)
				{
					num16 = Mathf.Round(num16 / _pfv3RotateSnap) * _pfv3RotateSnap;
				}
				_pfv3PreviewEuler = _pfv3StartEulerAdd + PFV3_AxisMask(new Vector3(num16, num16, num16), _pfv3ActiveAxis);
			}
			else if (_pfv3ActiveAxis == 3)
			{
				float num17 = (current.mousePosition.y - _pfv3MouseStart.y) / 120f;
				float num18 = Mathf.Clamp(_pfv3StartScale.x + num17, 0.01f, 50f);
				if (flag2)
				{
					num18 = Mathf.Max(_pfv3ScaleSnap, Mathf.Round(num18 / _pfv3ScaleSnap) * _pfv3ScaleSnap);
				}
				_pfv3PreviewScale = new Vector3(num18, num18, num18);
			}
			else if (_pfv3ActiveAxis == 3)
			{
				_pfv3DragPlaneNormal = (((Object)(object)PFV3_Cam != (Object)null) ? ((Component)PFV3_Cam).transform.forward : Vector3.forward);
				if (PFV3_RaycastToPlane(current.mousePosition, _pfv3Pivot, _pfv3DragPlaneNormal, out var world4))
				{
					val4 = world4 - _pfv3Pivot;
					_pfv3ScaleStartRadius = Mathf.Max(0.001f, ((Vector3)(ref val4)).magnitude);
				}
				else
				{
					Bounds bounds2 = _pfv3Sel.Bounds;
					val4 = ((Bounds)(ref bounds2)).extents;
					_pfv3ScaleStartRadius = Mathf.Max(0.001f, ((Vector3)(ref val4)).magnitude);
				}
				_pfv3StartScale = _pfv3PreviewScale;
			}
			else
			{
				Vector3? val11 = PFV3_ClosestPointOnAxis(_pfv3Pivot, _pfv3ActiveAxis);
				if (val11.HasValue)
				{
					Vector3 val12 = ((_pfv3ActiveAxis == 0) ? Vector3.right : ((_pfv3ActiveAxis == 1) ? Vector3.up : Vector3.forward));
					float num19 = Vector3.Dot(val11.Value - _pfv3DragStartWorld, ((Vector3)(ref val12)).normalized);
					float num20 = Mathf.Max(0.05f, _pfv3ScaleRefLen);
					float num21 = Mathf.Clamp(((Vector3)(ref _pfv3StartScale))[_pfv3ActiveAxis] + num19 / num20, 0.01f, 50f);
					if (flag2)
					{
						num21 = Mathf.Max(_pfv3ScaleSnap, Mathf.Round(num21 / _pfv3ScaleSnap) * _pfv3ScaleSnap);
					}
					_pfv3PreviewScale = _pfv3StartScale;
					((Vector3)(ref _pfv3PreviewScale))[_pfv3ActiveAxis] = num21;
				}
			}
			PFV3_ApplyPreviewLocally();
			current.Use();
		}
		if (_pfv3Dragging && _pfv3ActiveAxis == 3)
		{
			float num22 = Event.current.mousePosition.y - _pfv3MouseStart.y;
			float num23 = 0.0065f;
			float num24 = 1f - num22 * num23;
			float num25 = Mathf.Round(Mathf.Max(0.001f, _pfv3StartScale.x * num24) / _pfv3ScaleSnap) * _pfv3ScaleSnap;
			_pfv3PreviewScale = Vector3.one * Mathf.Max(0.001f, num25);
			PFV3_ApplyPreviewLocally();
			Event.current.Use();
		}
		if ((int)current.type != 1 || current.button != 0 || !_pfv3Dragging)
		{
			return;
		}
		_pfv3Dragging = false;
		_pfv3ActiveAxis = -1;
		_pfv3HaveEulerSent = false;
		current.Use();
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (!((Object)item.Tr == (Object)null))
			{
				item.Pos = item.Tr.position;
				item.Rot = item.Tr.rotation;
				item.Scale = item.Tr.localScale;
			}
		}
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
		_pfv3Pivot = PFV3_ComputePivot();
		return;
		IL_0b7e:
		int num26 = _pfv3Sel.Items.FindIndex(delegate(PFV3Captured x)
		{
			//IL_000f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_001a: Unknown result type (might be due to invalid IL or missing references)
			//IL_001f: Unknown result type (might be due to invalid IL or missing references)
			if (x.Hash == hash)
			{
				Vector3 val13 = x.Pos - pos;
				return ((Vector3)(ref val13)).sqrMagnitude < 1E-05f;
			}
			return false;
		});
		if (num2 != 0 && num26 >= 0)
		{
			_pfv3Sel.Items.RemoveAt(num26);
		}
		else
		{
			string name2 = PFV3_LeadingDigits(name) ?? hash.ToString();
			_pfv3Sel.Items.Add(new PFV3Captured
			{
				Name = name2,
				Hash = hash,
				Pos = pos,
				Rot = rot,
				Scale = val3.transform.localScale,
				Tr = val3.transform
			});
		}
		_pfv3Pivot = PFV3_ComputePivot();
		_pfv3SelFocus = ((_pfv3Sel.Items.Count > 0) ? Mathf.Clamp(_pfv3SelFocus, 0, _pfv3Sel.Items.Count - 1) : (-1));
		if (_pfv3SelFocus < 0 && _pfv3Sel.Items.Count > 0)
		{
			_pfv3SelFocus = 0;
		}
		goto IL_0c96;
		IL_0c96:
		_pfv3Pivot = PFV3_ComputePivot();
		_pfv3Sel.Phase = ((_pfv3Sel.Items.Count > 0) ? PFV3SelPhase.Selected : PFV3SelPhase.Idle);
		_pfv3Sel.Active = _pfv3Sel.Items.Count > 0;
		current.Use();
		goto IL_0187;
		IL_0ced:
		_pfv3Sel.Items.Clear();
		goto IL_0b7e;
	}

	private Vector3 PFV3_GetAxisDir(int axis)
	{
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0052: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_005d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_0078: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Unknown result type (might be due to invalid IL or missing references)
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_009b: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_0092: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_0087: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0091: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_006b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0070: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		if (!_pfv3LocalAxes)
		{
			return (Vector3)(axis switch
			{
				1 => Vector3.up, 
				0 => Vector3.right, 
				_ => Vector3.forward, 
			});
		}
		if (_pfv3Sel.Items.Count > 0)
		{
			Quaternion rot = _pfv3Sel.Items[0].Rot;
			Quaternion val = Quaternion.Euler(_pfv3PreviewEuler) * rot;
			return (Vector3)(axis switch
			{
				1 => val * Vector3.up, 
				0 => val * Vector3.right, 
				_ => val * Vector3.forward, 
			});
		}
		return (Vector3)(axis switch
		{
			1 => Vector3.up, 
			0 => Vector3.right, 
			_ => Vector3.forward, 
		});
	}

	private void PFV3_CaptureInBounds(Bounds b)
	{
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_004c: Expected O, but got Unknown
		//IL_0288: Unknown result type (might be due to invalid IL or missing references)
		//IL_028d: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_018e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_0227: Unknown result type (might be due to invalid IL or missing references)
		//IL_022c: Unknown result type (might be due to invalid IL or missing references)
		//IL_023e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0243: Unknown result type (might be due to invalid IL or missing references)
		//IL_0255: Unknown result type (might be due to invalid IL or missing references)
		//IL_025a: Unknown result type (might be due to invalid IL or missing references)
		_pfv3Sel.Items.Clear();
		foreach (PFV3Captured item in _pfv3Scan)
		{
			if (((Bounds)(ref b)).Contains(item.Pos))
			{
				Transform val = (((Object)item.Tr != (Object)null) ? item.Tr : PFV3_FindTransformFor(item));
				_pfv3Sel.Items.Add(new PFV3Captured
				{
					Display = item.Display,
					Id = (PFV3_LeadingDigits(item.Display) ?? item.Hash.ToString()),
					Hash = item.Hash,
					Pos = (((Object)(object)val != (Object)null) ? val.position : item.Pos),
					Rot = (((Object)(object)val != (Object)null) ? val.rotation : item.Rot),
					Scale = (((Object)(object)val != (Object)null) ? val.localScale : item.Scale),
					Tr = val
				});
			}
		}
		GameObject[] array = Object.FindObjectsOfType<GameObject>();
		foreach (GameObject go in array)
		{
			if (!go.activeInHierarchy)
			{
				continue;
			}
			NetworkPrefab val2 = go.GetComponent<NetworkPrefab>() ?? go.GetComponentInParent<NetworkPrefab>();
			if ((Object)(object)val2 != (Object)null && ((Bounds)(ref b)).Contains(go.transform.position))
			{
				uint h = val2.Hash;
				string value;
				string name = (_pfv3Hash2Name.TryGetValue(h, out value) ? value : ((Object)go).name);
				if (!_pfv3Sel.Items.Any(delegate(PFV3Captured x)
				{
					//IL_000f: Unknown result type (might be due to invalid IL or missing references)
					//IL_001f: Unknown result type (might be due to invalid IL or missing references)
					//IL_0024: Unknown result type (might be due to invalid IL or missing references)
					//IL_0029: Unknown result type (might be due to invalid IL or missing references)
					if (x.Hash == h)
					{
						Vector3 val3 = x.Pos - go.transform.position;
						return ((Vector3)(ref val3)).sqrMagnitude < 1E-05f;
					}
					return false;
				}))
				{
					_pfv3Sel.Items.Add(new PFV3Captured
					{
						Name = name,
						Hash = h,
						Pos = go.transform.position,
						Rot = go.transform.rotation,
						Scale = go.transform.localScale,
						Tr = go.transform
					});
				}
			}
		}
		_pfv3Pivot = PFV3_ComputePivot();
		_pfv3Sel.Active = _pfv3Sel.Items.Count > 0;
	}

	private PFV3ItemState PFV3_CaptureItemState(PFV3Captured c)
	{
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0094: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0099: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		Transform val = (((Object)(object)c.Tr != (Object)null) ? c.Tr : (c.Tr = PFV3_FindTransformFor(c)));
		return new PFV3ItemState
		{
			Name = c.Name,
			Display = c.Display,
			Id = c.Id,
			Hash = c.Hash,
			Pos = (((Object)(object)val != (Object)null) ? val.position : c.Pos),
			Rot = (((Object)(object)val != (Object)null) ? val.rotation : c.Rot),
			Scale = (((Object)(object)val != (Object)null) ? val.localScale : c.Scale),
			Tr = val
		};
	}

	private static float PFV3_ReadScrollY()
	{
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		Mouse current = Mouse.current;
		if (current == null)
		{
			return 0f;
		}
		return ((InputControl<Vector2>)(object)current.scroll).ReadValue().y;
	}

	private void PFV3_ApplyPreviewLocally()
	{
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0080: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_0088: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0091: Unknown result type (might be due to invalid IL or missing references)
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_009c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_00af: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fa: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ff: Unknown result type (might be due to invalid IL or missing references)
		//IL_0104: Unknown result type (might be due to invalid IL or missing references)
		//IL_0106: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_010b: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0111: Unknown result type (might be due to invalid IL or missing references)
		//IL_0116: Unknown result type (might be due to invalid IL or missing references)
		//IL_011b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0120: Unknown result type (might be due to invalid IL or missing references)
		//IL_0124: Unknown result type (might be due to invalid IL or missing references)
		//IL_0129: Unknown result type (might be due to invalid IL or missing references)
		//IL_012d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0137: Unknown result type (might be due to invalid IL or missing references)
		//IL_0185: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		Quaternion val = Quaternion.Euler(_pfv3PreviewEuler);
		Vector3 pfv3Pivot = _pfv3Pivot;
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			Transform val2 = (((Object)(object)item.Tr != (Object)null) ? item.Tr : (item.Tr = PFV3_FindTransformFor(item)));
			if ((Object)(object)val2 != (Object)null)
			{
				Vector3 val3 = item.Pos - pfv3Pivot;
				Vector3 val4 = pfv3Pivot + val * val3 + _pfv3PreviewOffset;
				Quaternion val5 = val * item.Rot;
				if ((Object)(object)val2.parent != (Object)null)
				{
					val2.localPosition = val2.parent.InverseTransformPoint(val4);
				}
				else
				{
					val2.localPosition = val4;
				}
				Quaternion val6 = (((Object)(object)val2.parent != (Object)null) ? (Quaternion.Inverse(val2.parent.rotation) * val5) : val5);
				Quaternion val7 = val6 * Quaternion.Inverse(val2.localRotation);
				Vector3 eulerAngles = ((Quaternion)(ref val7)).eulerAngles;
				val2.Rotate(eulerAngles, (Space)1);
				val2.localRotation = val6;
				val2.localScale = new Vector3(item.Scale.x * _pfv3PreviewScale.x, item.Scale.y * _pfv3PreviewScale.y, item.Scale.z * _pfv3PreviewScale.z);
				val2.hasChanged = true;
			}
		}
	}

	private bool PFV3_RaycastToPlane(Vector2 screen, Vector3 planePoint, Vector3 planeNormal, out Vector3 world)
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		world = default(Vector3);
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return false;
		}
		Ray val = pFV3_Cam.ScreenPointToRay(new Vector3(screen.x, screen.y, 0f));
		Plane val2 = default(Plane);
		((Plane)(ref val2))._002Ector(planeNormal, planePoint);
		float num = 0f;
		if (((Plane)(ref val2)).Raycast(val, ref num))
		{
			world = ((Ray)(ref val)).GetPoint(num);
			return true;
		}
		return false;
	}

	private bool PFV3_IsOverTopBar(Vector2 mouseBL)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		return (float)Screen.height - mouseBL.y <= 68f;
	}

	private bool PFV3_IsOverRightPanel(Vector2 mouseBL)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_000e: Unknown result type (might be due to invalid IL or missing references)
		float num = (float)Screen.height - mouseBL.y;
		if (mouseBL.x >= (float)Screen.width - 360f)
		{
			return num >= 68f;
		}
		return false;
	}

	private bool PFV3_Block3DInput(Vector2 mouseBL)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		if (!PFV3_IsOverTopBar(mouseBL))
		{
			return PFV3_IsOverRightPanel(mouseBL);
		}
		return true;
	}

	private void PFV3_DrawOverlays()
	{
		//IL_007f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0084: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_009c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0037: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_012d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0143: Unknown result type (might be due to invalid IL or missing references)
		//IL_015c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_02bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_02da: Unknown result type (might be due to invalid IL or missing references)
		//IL_02fd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0320: Unknown result type (might be due to invalid IL or missing references)
		//IL_0343: Unknown result type (might be due to invalid IL or missing references)
		//IL_0366: Unknown result type (might be due to invalid IL or missing references)
		//IL_021c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0221: Unknown result type (might be due to invalid IL or missing references)
		//IL_0226: Unknown result type (might be due to invalid IL or missing references)
		//IL_022c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0231: Unknown result type (might be due to invalid IL or missing references)
		//IL_0238: Unknown result type (might be due to invalid IL or missing references)
		//IL_039e: Unknown result type (might be due to invalid IL or missing references)
		//IL_03a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_027a: Unknown result type (might be due to invalid IL or missing references)
		//IL_025f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0493: Unknown result type (might be due to invalid IL or missing references)
		//IL_0415: Unknown result type (might be due to invalid IL or missing references)
		//IL_0417: Unknown result type (might be due to invalid IL or missing references)
		//IL_03b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_0293: Unknown result type (might be due to invalid IL or missing references)
		//IL_049f: Unknown result type (might be due to invalid IL or missing references)
		//IL_049b: Unknown result type (might be due to invalid IL or missing references)
		//IL_042a: Unknown result type (might be due to invalid IL or missing references)
		//IL_03bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_03bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_03af: Unknown result type (might be due to invalid IL or missing references)
		//IL_04a9: Unknown result type (might be due to invalid IL or missing references)
		//IL_0434: Unknown result type (might be due to invalid IL or missing references)
		//IL_0436: Unknown result type (might be due to invalid IL or missing references)
		//IL_0426: Unknown result type (might be due to invalid IL or missing references)
		//IL_03d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_04bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_04bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_044b: Unknown result type (might be due to invalid IL or missing references)
		//IL_03da: Unknown result type (might be due to invalid IL or missing references)
		//IL_03dc: Unknown result type (might be due to invalid IL or missing references)
		//IL_03ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_0455: Unknown result type (might be due to invalid IL or missing references)
		//IL_0457: Unknown result type (might be due to invalid IL or missing references)
		//IL_0447: Unknown result type (might be due to invalid IL or missing references)
		//IL_03f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_046c: Unknown result type (might be due to invalid IL or missing references)
		//IL_03ed: Unknown result type (might be due to invalid IL or missing references)
		//IL_0468: Unknown result type (might be due to invalid IL or missing references)
		//IL_04ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_04de: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)(object)_pfv3HoverTr != (Object)null)
		{
			Collider component = ((Component)_pfv3HoverTr).GetComponent<Collider>();
			if ((Object)(object)component != (Object)null)
			{
				Bounds bounds = component.bounds;
				PFV3_DrawAabb(((Bounds)(ref bounds)).center, ((Bounds)(ref bounds)).size, new Color(1f, 1f, 0f, 0.06f), new Color(1f, 1f, 0.25f, 0.45f));
			}
			Vector3 val = PFV3_Cam.WorldToScreenPoint(_pfv3HoverTr.position);
			Rect val2 = new Rect(val.x + 12f, (float)Screen.height - val.y - 10f, 420f, 20f);
			GUI.color = new Color(0f, 0f, 0f, 0.5f);
			GUI.Box(val2, GUIContent.none);
			GUI.color = Color.white;
			GUI.Label(val2, _pfv3HoverName ?? ((Object)_pfv3HoverTr).name, _small);
		}
		if (_pfv3ShowSelectionBox && _pfv3Sel.Active)
		{
			PFV3_GetSelectionBounds(out var center, out var size);
			PFV3_DrawAabb(center, size, new Color(0.2f, 0.8f, 1f, 0.08f), new Color(0.2f, 0.8f, 1f, 0.6f));
		}
		if (_pfv3ShowRuler)
		{
			PFV3_DrawRuler();
		}
		if (!_pfv3EditorOn)
		{
			return;
		}
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return;
		}
		if (_pfv3Sel.Active)
		{
			Vector3 val3 = ((Component)pFV3_Cam).transform.position - _pfv3Pivot;
			float magnitude = ((Vector3)(ref val3)).magnitude;
			_pfv3GizmoSize = Mathf.Clamp(magnitude * 0.12f, 0.35f, 6f);
		}
		if (_pfv3Sel.Active && (_pfv3Sel.Phase == PFV3SelPhase.DraggingRect || _pfv3Sel.Phase == PFV3SelPhase.Selected || _pfv3Sel.Phase == PFV3SelPhase.Preview))
		{
			Bounds bounds2 = _pfv3Sel.Bounds;
			PFV3_DrawAabb(((Bounds)(ref bounds2)).center + _pfv3PreviewOffset, ((Bounds)(ref bounds2)).size, (_pfv3Sel.Phase == PFV3SelPhase.Preview) ? new Color(0.2f, 0.8f, 1f, 0.16f) : new Color(0.2f, 0.6f, 1f, 0.1f), new Color(1f, 1f, 1f, 0.18f));
		}
		if (_pfv3Sel.Active)
		{
			PFV3_EnsureLineMat();
			_pfv3LineMat.SetPass(0);
			Vector3 pfv3Pivot = _pfv3Pivot;
			int num = PFV3_PickAxisHandle(pfv3Pivot);
			int pfv3ActiveAxis = _pfv3ActiveAxis;
			Color val4 = default(Color);
			((Color)(ref val4))._002Ector(1f, 0.25f, 0.25f, 1f);
			Color val5 = default(Color);
			((Color)(ref val5))._002Ector(0.3f, 1f, 0.35f, 1f);
			Color val6 = default(Color);
			((Color)(ref val6))._002Ector(0.3f, 0.65f, 1f, 1f);
			Color val7 = default(Color);
			((Color)(ref val7))._002Ector(1f, 1f, 0.25f, 1f);
			Color val8 = default(Color);
			((Color)(ref val8))._002Ector(0.95f, 0.95f, 0.95f, 1f);
			float length = 1.45f * _pfv3GizmoSize;
			if (_pfv3Mode == PFV3Mode.Translate)
			{
				PFV3_DrawAxisGizmo(pfv3Pivot, Vector3.right, length, (pfv3ActiveAxis == 0 || num == 0) ? val7 : val4);
				PFV3_DrawAxisGizmo(pfv3Pivot, Vector3.up, length, (pfv3ActiveAxis == 1 || num == 1) ? val7 : val5);
				PFV3_DrawAxisGizmo(pfv3Pivot, Vector3.forward, length, (pfv3ActiveAxis == 2 || num == 2) ? val7 : val6);
			}
			else if (_pfv3Mode == PFV3Mode.Rotate)
			{
				float radius = 1.2f * _pfv3GizmoSize;
				PFV3_DrawRing(pfv3Pivot, Vector3.right, radius, (pfv3ActiveAxis == 0 || num == 0) ? val7 : val4);
				PFV3_DrawRing(pfv3Pivot, Vector3.up, radius, (pfv3ActiveAxis == 1 || num == 1) ? val7 : val5);
				PFV3_DrawRing(pfv3Pivot, Vector3.forward, radius, (pfv3ActiveAxis == 2 || num == 2) ? val7 : val6);
			}
			else
			{
				float radius2 = 1.2f * _pfv3GizmoSize * 0.95f;
				bool flag = pfv3ActiveAxis == 3;
				PFV3_DrawCameraFacingRing(pfv3Pivot, radius2, flag ? val7 : val8, 48);
				PFV3_DrawCenterCube(pfv3Pivot, 0.08f * _pfv3GizmoSize, flag ? val7 : val8);
			}
			if (_pfv3ShowRuler)
			{
				PFV3_DrawRuler();
			}
			if (_pfv3ShowPlanes)
			{
				PFV3_DrawPlanes(_pfv3Pivot);
			}
			PFV3_DrawPlanes(_pfv3Pivot);
		}
	}

	private void PFV3_DrawPlanes(Vector3 pivot)
	{
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0070: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bf: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00de: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f4: Unknown result type (might be due to invalid IL or missing references)
		float S;
		Color w;
		if (_pfv3ShowPlanes)
		{
			S = _pfv3PlaneSize;
			Color val = default(Color);
			((Color)(ref val))._002Ector(0.2f, 0.6f, 1f, 0.1f);
			Color val2 = default(Color);
			((Color)(ref val2))._002Ector(0.6f, 1f, 0.2f, 0.1f);
			Color val3 = default(Color);
			((Color)(ref val3))._002Ector(1f, 0.6f, 0.2f, 0.1f);
			w = new Color(1f, 1f, 1f, 0.18f);
			Vector3 right = Vector3.right;
			Vector3 up = Vector3.up;
			Vector3 forward = Vector3.forward;
			Grid(Vector3.forward, right, up, val);
			Grid(Vector3.up, right, forward, val2);
			Grid(Vector3.right, forward, up, val3);
		}
		void Grid(Vector3 n, Vector3 u, Vector3 v, Color val4)
		{
			//IL_000c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0015: Unknown result type (might be due to invalid IL or missing references)
			//IL_001a: Unknown result type (might be due to invalid IL or missing references)
			//IL_001b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0020: Unknown result type (might be due to invalid IL or missing references)
			//IL_0021: Unknown result type (might be due to invalid IL or missing references)
			//IL_002d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0032: Unknown result type (might be due to invalid IL or missing references)
			//IL_003e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0043: Unknown result type (might be due to invalid IL or missing references)
			//IL_0044: Unknown result type (might be due to invalid IL or missing references)
			//IL_0045: Unknown result type (might be due to invalid IL or missing references)
			//IL_0051: Unknown result type (might be due to invalid IL or missing references)
			//IL_0056: Unknown result type (might be due to invalid IL or missing references)
			//IL_0062: Unknown result type (might be due to invalid IL or missing references)
			//IL_0067: Unknown result type (might be due to invalid IL or missing references)
			//IL_0068: Unknown result type (might be due to invalid IL or missing references)
			//IL_0069: Unknown result type (might be due to invalid IL or missing references)
			//IL_0075: Unknown result type (might be due to invalid IL or missing references)
			//IL_007a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0086: Unknown result type (might be due to invalid IL or missing references)
			//IL_008b: Unknown result type (might be due to invalid IL or missing references)
			//IL_008c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0091: Unknown result type (might be due to invalid IL or missing references)
			//IL_0092: Unknown result type (might be due to invalid IL or missing references)
			//IL_009e: Unknown result type (might be due to invalid IL or missing references)
			//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
			//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ee: Unknown result type (might be due to invalid IL or missing references)
			//IL_00ef: Unknown result type (might be due to invalid IL or missing references)
			//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
			//IL_0100: Unknown result type (might be due to invalid IL or missing references)
			//IL_0105: Unknown result type (might be due to invalid IL or missing references)
			//IL_0107: Unknown result type (might be due to invalid IL or missing references)
			//IL_010c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0118: Unknown result type (might be due to invalid IL or missing references)
			//IL_011d: Unknown result type (might be due to invalid IL or missing references)
			//IL_0125: Unknown result type (might be due to invalid IL or missing references)
			//IL_012a: Unknown result type (might be due to invalid IL or missing references)
			//IL_012f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0131: Unknown result type (might be due to invalid IL or missing references)
			//IL_0136: Unknown result type (might be due to invalid IL or missing references)
			//IL_0142: Unknown result type (might be due to invalid IL or missing references)
			//IL_0147: Unknown result type (might be due to invalid IL or missing references)
			//IL_0148: Unknown result type (might be due to invalid IL or missing references)
			//IL_0154: Unknown result type (might be due to invalid IL or missing references)
			//IL_0159: Unknown result type (might be due to invalid IL or missing references)
			//IL_015e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0160: Unknown result type (might be due to invalid IL or missing references)
			//IL_0165: Unknown result type (might be due to invalid IL or missing references)
			//IL_0171: Unknown result type (might be due to invalid IL or missing references)
			//IL_0176: Unknown result type (might be due to invalid IL or missing references)
			//IL_017e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0183: Unknown result type (might be due to invalid IL or missing references)
			//IL_0188: Unknown result type (might be due to invalid IL or missing references)
			//IL_018a: Unknown result type (might be due to invalid IL or missing references)
			//IL_018f: Unknown result type (might be due to invalid IL or missing references)
			PFV3_Begin3D();
			GL.Begin(7);
			GL.Color(val4);
			GL.Vertex(pivot + (-u - v) * S);
			GL.Vertex(pivot + (u - v) * S);
			GL.Vertex(pivot + (u + v) * S);
			GL.Vertex(pivot + (-u + v) * S);
			GL.End();
			GL.Begin(1);
			GL.Color(w);
			int num = Mathf.Max(1, _pfv3PlaneGrid);
			for (int i = -num; i <= num; i++)
			{
				float num2 = (float)i / (float)num * S;
				GL.Vertex(pivot + -u * S + v * num2);
				GL.Vertex(pivot + u * S + v * num2);
				GL.Vertex(pivot + -v * S + u * num2);
				GL.Vertex(pivot + v * S + u * num2);
			}
			GL.End();
			PFV3_End3D();
		}
	}

	private PFV3Snapshot PFV3_CaptureSnapshot()
	{
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		PFV3PreviewTx pFV3PreviewTx = new PFV3PreviewTx
		{
			Offset = _pfv3PreviewOffset,
			EulerAdd = _pfv3PreviewEuler,
			ScaleMul = _pfv3PreviewScale,
			Pivot = _pfv3Pivot
		};
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			pFV3PreviewTx.Src.Add(PFV3_CaptureItemState(item));
		}
		return new PFV3Snapshot(pFV3PreviewTx);
	}

	private bool PFV3_Raycast(Vector2 screenPos, out NetworkPrefab prefab, out RaycastHit hit)
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0015: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_005d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Expected O, but got Unknown
		if (Physics.Raycast((PFV3_Cam ?? Camera.main).ScreenPointToRay(new Vector3(screenPos.x, screenPos.y, 0f)), ref hit, 5000f, _pfv3RayMask))
		{
			prefab = (((Object)(object)((RaycastHit)(ref hit)).transform != (Object)null) ? ((Component)((RaycastHit)(ref hit)).transform).GetComponentInParent<NetworkPrefab>() : null);
			return (Object)prefab != (Object)null;
		}
		prefab = null;
		return false;
	}

	private bool PFV3_Raycast(Vector2 screenPos, out RaycastHit hit)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		NetworkPrefab prefab;
		return PFV3_Raycast(screenPos, out prefab, out hit);
	}

	private void PFV3_DrawBrushSection()
	{
		//IL_0135: Unknown result type (might be due to invalid IL or missing references)
		//IL_0140: Unknown result type (might be due to invalid IL or missing references)
		GUILayout.Space(10f);
		GUILayout.Label("", _h2, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		bool flag = GUILayout.Toggle(brushModeActive, brushModeActive ? "Enabled" : "Disabled", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
		if (flag != brushModeActive)
		{
			brushModeActive = flag;
			if (!brushModeActive)
			{
				if ((Object)(object)brushSphere != (Object)null)
				{
					brushSphere.SetActive(false);
				}
				ClearAllHighlights();
			}
		}
		GUILayout.FlexibleSpace();
		GUILayout.Label($"Size {brushSize:0.0}", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) });
		float num = GUILayout.HorizontalSlider(brushSize, 0.5f, 20f, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(160f) });
		if (Mathf.Abs(num - brushSize) > 0.0001f)
		{
			brushSize = num;
			if ((Object)(object)brushSphere != (Object)null)
			{
				brushSphere.transform.localScale = Vector3.one * brushSize;
			}
		}
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Clear Highlights", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			ClearAllHighlights();
		}
		GUILayout.EndHorizontal();
	}

	private void PFV3_ApplySnapshot(PFV3Snapshot snap)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_002a: Unknown result type (might be due to invalid IL or missing references)
		//IL_002f: Unknown result type (might be due to invalid IL or missing references)
		//IL_009c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b9: Unknown result type (might be due to invalid IL or missing references)
		if (snap == null)
		{
			return;
		}
		_pfv3PreviewOffset = snap.Offset;
		_pfv3PreviewEuler = snap.EulerAdd;
		_pfv3PreviewScale = snap.ScaleMul;
		_pfv3Pivot = snap.Pivot;
		_pfv3Sel.Items.Clear();
		foreach (PFV3ItemState item in snap.Src)
		{
			_pfv3Sel.Items.Add(new PFV3Captured
			{
				Name = item.Name,
				Display = item.Display,
				Id = item.Id,
				Hash = item.Hash,
				Pos = item.Pos,
				Rot = item.Rot,
				Scale = item.Scale,
				Tr = item.Tr
			});
		}
		_pfv3Sel.Active = _pfv3Sel.Items.Count > 0;
		_pfv3Sel.Phase = (_pfv3Sel.Active ? PFV3SelPhase.Selected : PFV3SelPhase.Idle);
		PFV3_ApplyPreviewLocally();
	}

	private void PFV3_GetSelectionBounds(out Vector3 center, out Vector3 size)
	{
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0055: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		//IL_0094: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ca: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_01df: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fa: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0102: Unknown result type (might be due to invalid IL or missing references)
		//IL_0108: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0112: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0122: Unknown result type (might be due to invalid IL or missing references)
		//IL_0130: Unknown result type (might be due to invalid IL or missing references)
		//IL_0172: Unknown result type (might be due to invalid IL or missing references)
		//IL_0174: Unknown result type (might be due to invalid IL or missing references)
		//IL_017e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0183: Unknown result type (might be due to invalid IL or missing references)
		//IL_0188: Unknown result type (might be due to invalid IL or missing references)
		//IL_018a: Unknown result type (might be due to invalid IL or missing references)
		//IL_018d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0192: Unknown result type (might be due to invalid IL or missing references)
		//IL_0197: Unknown result type (might be due to invalid IL or missing references)
		//IL_0198: Unknown result type (might be due to invalid IL or missing references)
		//IL_019c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a6: Unknown result type (might be due to invalid IL or missing references)
		//IL_0169: Unknown result type (might be due to invalid IL or missing references)
		//IL_014f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0151: Unknown result type (might be due to invalid IL or missing references)
		//IL_015b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0160: Unknown result type (might be due to invalid IL or missing references)
		//IL_016e: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3Sel.Items.Count == 0)
		{
			center = Vector3.zero;
			size = Vector3.zero;
			return;
		}
		bool flag = _pfv3SpawnPreviewActive || _pfv3Sel.Phase == PFV3SelPhase.Preview;
		Quaternion val = Quaternion.Euler(_pfv3PreviewEuler);
		Vector3 pfv3Pivot = _pfv3Pivot;
		Vector3 val2 = default(Vector3);
		((Vector3)(ref val2))._002Ector(float.PositiveInfinity, float.PositiveInfinity, float.PositiveInfinity);
		Vector3 val3 = default(Vector3);
		((Vector3)(ref val3))._002Ector(float.NegativeInfinity, float.NegativeInfinity, float.NegativeInfinity);
		Bounds val4 = default(Bounds);
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			Transform val5 = (((Object)(object)item.Tr != (Object)null) ? item.Tr : (item.Tr = PFV3_FindTransformFor(item)));
			Vector3 val7;
			if (flag)
			{
				Vector3 val6 = item.Pos - pfv3Pivot;
				val7 = pfv3Pivot + val * val6 + _pfv3PreviewOffset;
			}
			else
			{
				val7 = (((Object)(object)val5 != (Object)null) ? val5.position : item.Pos);
			}
			if ((Object)(object)val5 != (Object)null)
			{
				Collider component = ((Component)val5).GetComponent<Collider>();
				val4 = (Bounds)(((Object)(object)component != (Object)null) ? component.bounds : new Bounds(val7, Vector3.one * 0.25f));
			}
			else
			{
				val4 = new Bounds(val7, Vector3.one * 0.25f);
			}
			val2 = Vector3.Min(val2, ((Bounds)(ref val4)).min);
			val3 = Vector3.Max(val3, ((Bounds)(ref val4)).max);
		}
		center = 0.5f * (val2 + val3);
		size = val3 - val2;
	}

	private void PFV3_ConfirmSpawnFromPreview()
	{
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		//IL_005d: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_0066: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0074: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Unknown result type (might be due to invalid IL or missing references)
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0080: Unknown result type (might be due to invalid IL or missing references)
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_0087: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_012f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0134: Unknown result type (might be due to invalid IL or missing references)
		//IL_013a: Unknown result type (might be due to invalid IL or missing references)
		//IL_013f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0145: Unknown result type (might be due to invalid IL or missing references)
		//IL_014a: Unknown result type (might be due to invalid IL or missing references)
		if (!_pfv3SpawnPreviewActive || _pfv3Sel.Items.Count == 0)
		{
			return;
		}
		Quaternion val = Quaternion.Euler(_pfv3PreviewEuler);
		Vector3 pfv3Pivot = _pfv3Pivot;
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			if (item.Hash != 0)
			{
				Vector3 val2 = item.Pos - pfv3Pivot;
				Vector3 pos = pfv3Pivot + val * val2 + _pfv3PreviewOffset;
				Quaternion rot = val * item.Rot;
				float uniformScale = Mathf.Max(0.0001f, (item.Scale.x + item.Scale.y + item.Scale.z) / 3f) * Mathf.Max(0.0001f, _pfv3PreviewScale.x);
				string text = PFV3_BuildSpawnCsv(item.Hash, pos, rot, uniformScale, _pfv3Kinematic);
				SendConsoleCmd("spawn string-raw " + text);
			}
		}
		_pfv3SpawnPreviewActive = false;
		_pfv3SpawnPreviewFile = null;
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
		_pfv3Sel.Items.Clear();
		_pfv3Sel.Active = false;
		_pfv3Sel.Phase = PFV3SelPhase.Idle;
		_statusLeft = "Spawned to server.";
	}

	private void PFV3_DrawRuler()
	{
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f8: Unknown result type (might be due to invalid IL or missing references)
		//IL_0102: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3RulerPts.Count != 0)
		{
			float num = 0f;
			for (int i = 0; i < _pfv3RulerPts.Count - 1; i++)
			{
				Vector3 val = _pfv3RulerPts[i];
				Vector3 val2 = _pfv3RulerPts[i + 1];
				PFV3_DrawBillboardSegment(val, val2, new Color(1f, 1f, 1f, 0.85f), 0.02f);
				num += Vector3.Distance(val, val2);
			}
			Camera pFV3_Cam = PFV3_Cam;
			if ((Object)(object)pFV3_Cam != (Object)null)
			{
				Vector3 val3 = _pfv3RulerPts[_pfv3RulerPts.Count - 1];
				Vector3 val4 = pFV3_Cam.WorldToScreenPoint(val3);
				Rect val5 = new Rect(val4.x + 10f, (float)Screen.height - val4.y - 10f, 200f, 20f);
				GUI.color = new Color(0f, 0f, 0f, 0.5f);
				GUI.Box(val5, GUIContent.none);
				GUI.color = Color.white;
				GUI.Label(val5, $"Total: {num:0.###} m", _small);
			}
		}
	}

	private void PFV3_DrawBillboardSegment(Vector3 a, Vector3 b, Color color, float thickness = 1.5f)
	{
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0034: Unknown result type (might be due to invalid IL or missing references)
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		PFV3_EnsureLineMat();
		if ((Object)(object)PFV3_Cam != (Object)null)
		{
			_pfv3LineMat.SetPass(0);
			GL.PushMatrix();
			GL.MultMatrix(Matrix4x4.identity);
			GL.Begin(1);
			GL.Color(color);
			GL.Vertex(a);
			GL.Vertex(b);
			GL.End();
			GL.PopMatrix();
		}
	}

	private void PFV3_DrawBillboardSegment(Vector3 center, Vector3 normal, float radius, float startDeg, float endDeg, Color color, float thickness = 0.02f, int segments = 48)
	{
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_0034: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0056: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_0066: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_006c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ff: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		//IL_0108: Unknown result type (might be due to invalid IL or missing references)
		//IL_010d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0115: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0120: Unknown result type (might be due to invalid IL or missing references)
		//IL_0125: Unknown result type (might be due to invalid IL or missing references)
		//IL_012a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0131: Unknown result type (might be due to invalid IL or missing references)
		//IL_0137: Unknown result type (might be due to invalid IL or missing references)
		PFV3_EnsureLineMat();
		if ((Object)(object)PFV3_Cam != (Object)null)
		{
			Vector3 val = ((((Vector3)(ref normal)).sqrMagnitude < 1E-06f) ? Vector3.up : ((Vector3)(ref normal)).normalized);
			Vector3 val2 = Vector3.Cross((Mathf.Abs(Vector3.Dot(val, Vector3.up)) > 0.9f) ? Vector3.right : Vector3.up, val);
			Vector3 normalized = ((Vector3)(ref val2)).normalized;
			Vector3 val3 = Vector3.Cross(val, normalized);
			float num = startDeg * ((float)Math.PI / 180f);
			float num2 = endDeg * ((float)Math.PI / 180f);
			if (num2 < num)
			{
				float num3 = num2;
				num2 = num;
				num = num3;
			}
			_pfv3LineMat.SetPass(0);
			GL.PushMatrix();
			GL.MultMatrix(Matrix4x4.identity);
			GL.Begin(1);
			GL.Color(color);
			Vector3 val4 = center + (normalized * Mathf.Cos(num) + val3 * Mathf.Sin(num)) * radius;
			for (int i = 1; i <= segments; i++)
			{
				float num4 = (float)i / (float)segments;
				float num5 = Mathf.Lerp(num, num2, num4);
				Vector3 val5 = center + (normalized * Mathf.Cos(num5) + val3 * Mathf.Sin(num5)) * radius;
				GL.Vertex(val4);
				GL.Vertex(val5);
				val4 = val5;
			}
			GL.End();
			GL.PopMatrix();
		}
	}

	private static string PFV3_LeadingDigits(string s)
	{
		if (string.IsNullOrEmpty(s))
		{
			return null;
		}
		Match match = Regex.Match(s, "^\\s*(\\d+)");
		if (!match.Success)
		{
			return null;
		}
		return match.Groups[1].Value;
	}

	private void PFV3_DrawAabb(Vector3 center, Vector3 size, Color fill, Color wire)
	{
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_003e: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0056: Unknown result type (might be due to invalid IL or missing references)
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0062: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_006d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0073: Unknown result type (might be due to invalid IL or missing references)
		//IL_0079: Unknown result type (might be due to invalid IL or missing references)
		//IL_007e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0084: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0090: Unknown result type (might be due to invalid IL or missing references)
		//IL_0095: Unknown result type (might be due to invalid IL or missing references)
		//IL_009b: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		//IL_00df: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fc: Unknown result type (might be due to invalid IL or missing references)
		//IL_0102: Unknown result type (might be due to invalid IL or missing references)
		//IL_0108: Unknown result type (might be due to invalid IL or missing references)
		//IL_0112: Unknown result type (might be due to invalid IL or missing references)
		//IL_0118: Unknown result type (might be due to invalid IL or missing references)
		//IL_011e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0124: Unknown result type (might be due to invalid IL or missing references)
		//IL_0129: Unknown result type (might be due to invalid IL or missing references)
		//IL_012f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0135: Unknown result type (might be due to invalid IL or missing references)
		//IL_013b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0140: Unknown result type (might be due to invalid IL or missing references)
		//IL_0146: Unknown result type (might be due to invalid IL or missing references)
		//IL_014c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0152: Unknown result type (might be due to invalid IL or missing references)
		//IL_0157: Unknown result type (might be due to invalid IL or missing references)
		//IL_015d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0163: Unknown result type (might be due to invalid IL or missing references)
		//IL_0169: Unknown result type (might be due to invalid IL or missing references)
		//IL_0173: Unknown result type (might be due to invalid IL or missing references)
		//IL_0179: Unknown result type (might be due to invalid IL or missing references)
		//IL_017f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0185: Unknown result type (might be due to invalid IL or missing references)
		//IL_018a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_0196: Unknown result type (might be due to invalid IL or missing references)
		//IL_019c: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01be: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ca: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01da: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e6: Unknown result type (might be due to invalid IL or missing references)
		//IL_01eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01fd: Unknown result type (might be due to invalid IL or missing references)
		//IL_0202: Unknown result type (might be due to invalid IL or missing references)
		//IL_0208: Unknown result type (might be due to invalid IL or missing references)
		//IL_020e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0214: Unknown result type (might be due to invalid IL or missing references)
		//IL_0219: Unknown result type (might be due to invalid IL or missing references)
		//IL_021f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0225: Unknown result type (might be due to invalid IL or missing references)
		//IL_022b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0235: Unknown result type (might be due to invalid IL or missing references)
		//IL_023b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0241: Unknown result type (might be due to invalid IL or missing references)
		//IL_0247: Unknown result type (might be due to invalid IL or missing references)
		//IL_024c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0252: Unknown result type (might be due to invalid IL or missing references)
		//IL_0258: Unknown result type (might be due to invalid IL or missing references)
		//IL_025e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0263: Unknown result type (might be due to invalid IL or missing references)
		//IL_0269: Unknown result type (might be due to invalid IL or missing references)
		//IL_026f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0275: Unknown result type (might be due to invalid IL or missing references)
		//IL_027a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0280: Unknown result type (might be due to invalid IL or missing references)
		//IL_0286: Unknown result type (might be due to invalid IL or missing references)
		//IL_028c: Unknown result type (might be due to invalid IL or missing references)
		//IL_02a6: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_02cc: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_02dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_02e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ef: Unknown result type (might be due to invalid IL or missing references)
		//IL_02f5: Unknown result type (might be due to invalid IL or missing references)
		//IL_02fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_0300: Unknown result type (might be due to invalid IL or missing references)
		//IL_030c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0312: Unknown result type (might be due to invalid IL or missing references)
		//IL_0318: Unknown result type (might be due to invalid IL or missing references)
		//IL_031e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0323: Unknown result type (might be due to invalid IL or missing references)
		//IL_032f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0335: Unknown result type (might be due to invalid IL or missing references)
		//IL_033b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0341: Unknown result type (might be due to invalid IL or missing references)
		//IL_0346: Unknown result type (might be due to invalid IL or missing references)
		//IL_0352: Unknown result type (might be due to invalid IL or missing references)
		//IL_0358: Unknown result type (might be due to invalid IL or missing references)
		//IL_035e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0364: Unknown result type (might be due to invalid IL or missing references)
		//IL_0369: Unknown result type (might be due to invalid IL or missing references)
		//IL_0375: Unknown result type (might be due to invalid IL or missing references)
		//IL_037b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0381: Unknown result type (might be due to invalid IL or missing references)
		//IL_0387: Unknown result type (might be due to invalid IL or missing references)
		//IL_038c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0398: Unknown result type (might be due to invalid IL or missing references)
		//IL_039e: Unknown result type (might be due to invalid IL or missing references)
		//IL_03a4: Unknown result type (might be due to invalid IL or missing references)
		//IL_03aa: Unknown result type (might be due to invalid IL or missing references)
		//IL_03af: Unknown result type (might be due to invalid IL or missing references)
		//IL_03bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_03c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_03c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_03cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_03d2: Unknown result type (might be due to invalid IL or missing references)
		PFV3_EnsureLineMat();
		_pfv3LineMat.SetPass(0);
		GL.PushMatrix();
		GL.MultMatrix(Matrix4x4.identity);
		GL.Begin(7);
		GL.Color(fill);
		Vector3 val = center - size * 0.5f;
		Vector3 val2 = center + size * 0.5f;
		Q(new Vector3(val.x, val2.y, val.z), new Vector3(val2.x, val2.y, val.z), new Vector3(val2.x, val2.y, val2.z), new Vector3(val.x, val2.y, val2.z));
		Q(new Vector3(val.x, val.y, val.z), new Vector3(val.x, val.y, val2.z), new Vector3(val2.x, val.y, val2.z), new Vector3(val2.x, val.y, val.z));
		Q(new Vector3(val.x, val.y, val.z), new Vector3(val.x, val2.y, val.z), new Vector3(val.x, val2.y, val2.z), new Vector3(val.x, val.y, val2.z));
		Q(new Vector3(val2.x, val.y, val.z), new Vector3(val2.x, val.y, val2.z), new Vector3(val2.x, val2.y, val2.z), new Vector3(val2.x, val2.y, val.z));
		Q(new Vector3(val.x, val.y, val2.z), new Vector3(val.x, val2.y, val2.z), new Vector3(val2.x, val2.y, val2.z), new Vector3(val2.x, val.y, val2.z));
		Q(new Vector3(val.x, val.y, val.z), new Vector3(val2.x, val.y, val.z), new Vector3(val2.x, val2.y, val.z), new Vector3(val.x, val2.y, val.z));
		GL.End();
		GL.PopMatrix();
		PFV3_Begin3D();
		GL.Color(wire);
		Vector3[] v = (Vector3[])(object)new Vector3[8];
		v[0] = new Vector3(val.x, val.y, val.z);
		v[1] = new Vector3(val2.x, val.y, val.z);
		v[2] = new Vector3(val2.x, val2.y, val.z);
		v[3] = new Vector3(val.x, val2.y, val.z);
		v[4] = new Vector3(val.x, val.y, val2.z);
		v[5] = new Vector3(val2.x, val.y, val2.z);
		v[6] = new Vector3(val2.x, val2.y, val2.z);
		v[7] = new Vector3(val.x, val2.y, val2.z);
		GL.Begin(1);
		L(0, 1);
		L(1, 2);
		L(2, 3);
		L(3, 0);
		L(4, 5);
		L(5, 6);
		L(6, 7);
		L(7, 4);
		L(0, 4);
		L(1, 5);
		L(2, 6);
		L(3, 7);
		GL.End();
		PFV3_End3D();
		void L(int a, int b)
		{
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0018: Unknown result type (might be due to invalid IL or missing references)
			GL.Vertex(v[a]);
			GL.Vertex(v[b]);
		}
		static void Q(Vector3 a, Vector3 b, Vector3 c, Vector3 d)
		{
			//IL_0000: Unknown result type (might be due to invalid IL or missing references)
			//IL_0006: Unknown result type (might be due to invalid IL or missing references)
			//IL_000c: Unknown result type (might be due to invalid IL or missing references)
			//IL_0012: Unknown result type (might be due to invalid IL or missing references)
			GL.Vertex(a);
			GL.Vertex(b);
			GL.Vertex(c);
			GL.Vertex(d);
		}
	}

	private void PFV3_DrawAxisGizmo(Vector3 origin, Vector3 dir, float length, Color col)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_000d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		GL.Begin(1);
		GL.Color(col);
		GL.Vertex(origin);
		GL.Vertex(origin + dir * (length * 0.82f));
		GL.End();
		PFV3_DrawCone(origin + dir * (length * 0.82f), dir, length * 0.18f, 0.06f * _pfv3GizmoSize, col);
	}

	private void PFV3_DrawRing(Vector3 center, Vector3 normal, float radius, Color col, int segs = 64)
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_000d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_004f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0067: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Unknown result type (might be due to invalid IL or missing references)
		//IL_007a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0080: Unknown result type (might be due to invalid IL or missing references)
		//IL_0085: Unknown result type (might be due to invalid IL or missing references)
		//IL_008a: Unknown result type (might be due to invalid IL or missing references)
		//IL_008f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0095: Unknown result type (might be due to invalid IL or missing references)
		//IL_009b: Unknown result type (might be due to invalid IL or missing references)
		GL.Begin(1);
		GL.Color(col);
		Quaternion val = Quaternion.FromToRotation(Vector3.forward, ((Vector3)(ref normal)).normalized);
		Vector3 val2 = center + val * (new Vector3(Mathf.Cos(0f), Mathf.Sin(0f), 0f) * radius);
		for (int i = 1; i <= segs; i++)
		{
			float num = (float)i / (float)segs * (float)Math.PI * 2f;
			Vector3 val3 = center + val * (new Vector3(Mathf.Cos(num), Mathf.Sin(num), 0f) * radius);
			GL.Vertex(val2);
			GL.Vertex(val3);
			val2 = val3;
		}
		GL.End();
	}

	private void PFV3_DrawCameraFacingRing(Vector3 center, float radius, Color col, int segs)
	{
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam != (Object)null)
		{
			PFV3_DrawRing(center, ((Component)pFV3_Cam).transform.forward, radius, col, segs);
		}
	}

	private void PFV3_DrawCenterCube(Vector3 c, float s, Color col)
	{
		//IL_0009: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Unknown result type (might be due to invalid IL or missing references)
		//IL_0061: Unknown result type (might be due to invalid IL or missing references)
		//IL_0066: Unknown result type (might be due to invalid IL or missing references)
		//IL_006b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0072: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_009c: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00de: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ea: Unknown result type (might be due to invalid IL or missing references)
		//IL_0100: Unknown result type (might be due to invalid IL or missing references)
		//IL_0105: Unknown result type (might be due to invalid IL or missing references)
		//IL_010a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0111: Unknown result type (might be due to invalid IL or missing references)
		//IL_0121: Unknown result type (might be due to invalid IL or missing references)
		//IL_0126: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0133: Unknown result type (might be due to invalid IL or missing references)
		//IL_013d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0142: Unknown result type (might be due to invalid IL or missing references)
		//IL_0147: Unknown result type (might be due to invalid IL or missing references)
		//IL_014f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0159: Unknown result type (might be due to invalid IL or missing references)
		//IL_015e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0163: Unknown result type (might be due to invalid IL or missing references)
		//IL_016b: Unknown result type (might be due to invalid IL or missing references)
		//IL_016f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0174: Unknown result type (might be due to invalid IL or missing references)
		//IL_0179: Unknown result type (might be due to invalid IL or missing references)
		//IL_0181: Unknown result type (might be due to invalid IL or missing references)
		//IL_0185: Unknown result type (might be due to invalid IL or missing references)
		//IL_018a: Unknown result type (might be due to invalid IL or missing references)
		//IL_018f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0197: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a6: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bd: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01cf: Unknown result type (might be due to invalid IL or missing references)
		//IL_01df: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e4: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_01f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_0207: Unknown result type (might be due to invalid IL or missing references)
		//IL_020c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0211: Unknown result type (might be due to invalid IL or missing references)
		//IL_0219: Unknown result type (might be due to invalid IL or missing references)
		//IL_0229: Unknown result type (might be due to invalid IL or missing references)
		//IL_022e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0233: Unknown result type (might be due to invalid IL or missing references)
		//IL_023b: Unknown result type (might be due to invalid IL or missing references)
		//IL_024b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0250: Unknown result type (might be due to invalid IL or missing references)
		//IL_0255: Unknown result type (might be due to invalid IL or missing references)
		//IL_025d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0267: Unknown result type (might be due to invalid IL or missing references)
		//IL_026c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0271: Unknown result type (might be due to invalid IL or missing references)
		//IL_0279: Unknown result type (might be due to invalid IL or missing references)
		//IL_0283: Unknown result type (might be due to invalid IL or missing references)
		//IL_0288: Unknown result type (might be due to invalid IL or missing references)
		//IL_028d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0295: Unknown result type (might be due to invalid IL or missing references)
		//IL_0299: Unknown result type (might be due to invalid IL or missing references)
		//IL_029e: Unknown result type (might be due to invalid IL or missing references)
		//IL_02a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_02ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_02bb: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c0: Unknown result type (might be due to invalid IL or missing references)
		//IL_02c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_02cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_02d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_02dc: Unknown result type (might be due to invalid IL or missing references)
		//IL_02e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_02f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_02fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_030c: Unknown result type (might be due to invalid IL or missing references)
		Vector3[] array = (Vector3[])(object)new Vector3[24]
		{
			c + new Vector3(0f - s, 0f - s, 0f - s),
			c + new Vector3(s, 0f - s, 0f - s),
			c + new Vector3(s, 0f - s, 0f - s),
			c + new Vector3(s, 0f - s, s),
			c + new Vector3(s, 0f - s, s),
			c + new Vector3(0f - s, 0f - s, s),
			c + new Vector3(0f - s, 0f - s, s),
			c + new Vector3(0f - s, 0f - s, 0f - s),
			c + new Vector3(0f - s, s, 0f - s),
			c + new Vector3(s, s, 0f - s),
			c + new Vector3(s, s, 0f - s),
			c + new Vector3(s, s, s),
			c + new Vector3(s, s, s),
			c + new Vector3(0f - s, s, s),
			c + new Vector3(0f - s, s, s),
			c + new Vector3(0f - s, s, 0f - s),
			c + new Vector3(0f - s, 0f - s, 0f - s),
			c + new Vector3(0f - s, s, 0f - s),
			c + new Vector3(s, 0f - s, 0f - s),
			c + new Vector3(s, s, 0f - s),
			c + new Vector3(s, 0f - s, s),
			c + new Vector3(s, s, s),
			c + new Vector3(0f - s, 0f - s, s),
			c + new Vector3(0f - s, s, s)
		};
		GL.Begin(1);
		GL.Color(col);
		for (int i = 0; i < array.Length; i += 2)
		{
			GL.Vertex(array[i]);
			GL.Vertex(array[i + 1]);
		}
		GL.End();
	}

	private void PFV3_DrawCone(Vector3 basePos, Vector3 dir, float length, float radius, Color col, int segs = 18)
	{
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0009: Unknown result type (might be due to invalid IL or missing references)
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0036: Unknown result type (might be due to invalid IL or missing references)
		//IL_0037: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_0046: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_0048: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0055: Unknown result type (might be due to invalid IL or missing references)
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		//IL_0097: Unknown result type (might be due to invalid IL or missing references)
		//IL_0098: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ad: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00db: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ec: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fd: Unknown result type (might be due to invalid IL or missing references)
		dir = ((Vector3)(ref dir)).normalized;
		Vector3 val = basePos + dir * length;
		Vector3 val2 = Vector3.up;
		if (Mathf.Abs(Vector3.Dot(val2, dir)) > 0.99f)
		{
			val2 = Vector3.right;
		}
		Vector3 val3 = Vector3.Cross(dir, val2);
		Vector3 normalized = ((Vector3)(ref val3)).normalized;
		val3 = Vector3.Cross(normalized, dir);
		val2 = ((Vector3)(ref val3)).normalized;
		GL.Begin(4);
		GL.Color(col);
		for (int i = 0; i < segs; i++)
		{
			float num = (float)i / (float)segs * (float)Math.PI * 2f;
			float num2 = (float)(i + 1) / (float)segs * (float)Math.PI * 2f;
			Vector3 val4 = basePos + (normalized * Mathf.Cos(num) + val2 * Mathf.Sin(num)) * radius;
			Vector3 val5 = basePos + (normalized * Mathf.Cos(num2) + val2 * Mathf.Sin(num2)) * radius;
			GL.Vertex(val4);
			GL.Vertex(val5);
			GL.Vertex(val);
		}
		GL.End();
	}

	private void PFV3_DrawRightPanel()
	{
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_05de: Unknown result type (might be due to invalid IL or missing references)
		//IL_05e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_05f3: Expected O, but got Unknown
		//IL_06b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_06ce: Unknown result type (might be due to invalid IL or missing references)
		//IL_06d3: Unknown result type (might be due to invalid IL or missing references)
		//IL_076e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0779: Unknown result type (might be due to invalid IL or missing references)
		//IL_0783: Expected O, but got Unknown
		//IL_07b4: Unknown result type (might be due to invalid IL or missing references)
		//IL_07d1: Unknown result type (might be due to invalid IL or missing references)
		//IL_07d6: Unknown result type (might be due to invalid IL or missing references)
		GUILayout.BeginArea(new Rect((float)Screen.width - 360f, 68f, 360f, (float)Screen.height - 68f), _card);
		GUILayout.Label("Actions", _h1, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Delete", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_DeleteSelected();
		}
		if (GUILayout.Button("Snap Ground", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_SnapGroundSelected();
		}
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Look at Me", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_SelectLookAtMe();
		}
		if (GUILayout.Button("Unselect", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_UnselectAll();
		}
		GUILayout.EndHorizontal();
		GUILayout.Space(6f);
		GUILayout.Label("PrefabV3 Bitch", Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Tool:", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
		int pfv3Tool = (int)_pfv3Tool;
		pfv3Tool = GUILayout.SelectionGrid(pfv3Tool, new string[3] { "Move", "Clone", "Stack" }, 3, Array.Empty<GUILayoutOption>());
		_pfv3Tool = (PFV3Tool)pfv3Tool;
		GUILayout.EndHorizontal();
		GUILayout.Space(8f);
		GUILayout.Label("Overlays", _h2, Array.Empty<GUILayoutOption>());
		_pfv3ShowSelectionBox = GUILayout.Toggle(_pfv3ShowSelectionBox, "Selection Box (AABB)", Array.Empty<GUILayoutOption>());
		_pfv3ShowRuler = GUILayout.Toggle(_pfv3ShowRuler, "Ruler (add points with Left-Click)", Array.Empty<GUILayoutOption>());
		_pfv3ShowPlanes = GUILayout.Toggle(_pfv3ShowPlanes, "Plane Gizmos (XY / XZ / YZ)", Array.Empty<GUILayoutOption>());
		if (_pfv3ShowRuler)
		{
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button("Clear Ruler", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(22f) }))
			{
				_pfv3RulerPts.Clear();
			}
			GUILayout.Label($"Pts: {_pfv3RulerPts.Count}", _small, Array.Empty<GUILayoutOption>());
			GUILayout.EndHorizontal();
		}
		GUILayout.Space(8f);
		GUILayout.Label("History", _h2, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		if (GUILayout.Button("Undo", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_Undo();
		}
		if (GUILayout.Button("Redo", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(24f) }))
		{
			PFV3_Redo();
		}
		GUILayout.EndHorizontal();
		_pfv3LocalAxes = GUILayout.Toggle(_pfv3LocalAxes, "Local Axes", Array.Empty<GUILayoutOption>());
		_pfv3KeepExisting = GUILayout.Toggle(_pfv3KeepExisting, "Clone", Array.Empty<GUILayoutOption>());
		_pfv3Kinematic = GUILayout.Toggle(_pfv3Kinematic, _pfv3Kinematic ? "Kinematic: ON" : "Kinematic: OFF", _btn, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Nudge:", (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(60f) });
		_pfv3NudgeStep = PFV3_FloatFieldInline(_pfv3NudgeStep, 64f);
		GUILayout.EndHorizontal();
		GUILayout.Space(6f);
		GUILayout.Label("Stack (Axiom style)", _h2, Array.Empty<GUILayoutOption>());
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Axis:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(40f) });
		int num = ((_pfv3KeyAxis >= 0) ? _pfv3KeyAxis : 0);
		num = (_pfv3KeyAxis = GUILayout.Toolbar(num, new string[3] { "X", "Y", "Z" }, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(120f) }));
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Copies:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(56f) });
		int num2 = Mathf.Clamp(PFV3_IntField(num switch
		{
			1 => _pfv3StackY, 
			0 => _pfv3StackX, 
			_ => _pfv3StackZ, 
		}, 1, 999), 1, 999);
		switch (num)
		{
		case 0:
			_pfv3StackX = num2;
			break;
		case 1:
			_pfv3StackY = num2;
			break;
		default:
			_pfv3StackZ = num2;
			break;
		}
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Spacing:", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(56f) });
		float num3 = Mathf.Max(0.001f, PFV3_FloatField(num switch
		{
			1 => _pfv3StackStep.y, 
			0 => _pfv3StackStep.x, 
			_ => _pfv3StackStep.z, 
		}));
		switch (num)
		{
		case 0:
			_pfv3StackStep.x = num3;
			break;
		case 1:
			_pfv3StackStep.y = num3;
			break;
		default:
			_pfv3StackStep.z = num3;
			break;
		}
		GUILayout.EndHorizontal();
		GUILayout.Space(4f);
		GUILayout.Label("Hints: [X/Y/Z] lock axis · Scroll = nudge · MMB = push · Ctrl=Snap · Shift=Fine · Ctrl+R=Rotate · Ctrl+F=Flip", Array.Empty<GUILayoutOption>());
		GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 6f), (Texture)_texDivider);
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Scanned Nearby", _h2, Array.Empty<GUILayoutOption>());
		GUILayout.FlexibleSpace();
		if (GUILayout.Button("Rescan", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(22f),
			GUILayout.Width(72f)
		}))
		{
			PFV3_DoScan();
		}
		GUILayout.EndHorizontal();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUILayout.Label("Filter", _small, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(40f) });
		_pfv3ScanFilter = GUILayout.TextField(_pfv3ScanFilter ?? "", Array.Empty<GUILayoutOption>());
		GUILayout.EndHorizontal();
		_pfv3ScanDropScroll = GUILayout.BeginScrollView(_pfv3ScanDropScroll, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(220f) });
		foreach (PFV3Captured item in _pfv3Scan)
		{
			if ((string.IsNullOrEmpty(_pfv3ScanFilter) || (item.Display != null && item.Display.IndexOf(_pfv3ScanFilter, StringComparison.OrdinalIgnoreCase) >= 0)) && GUILayout.Button(item.Display ?? "", _sidebarBtn, Array.Empty<GUILayoutOption>()))
			{
				PFV3_SelectSingle(item);
			}
		}
		GUILayout.EndScrollView();
		GUI.DrawTexture(GUILayoutUtility.GetRect(1f, 6f), (Texture)_texDivider);
		GUILayout.Label($"Selected ({_pfv3Sel.Items.Count})", _h2, Array.Empty<GUILayoutOption>());
		_pfv3SelDropScroll = GUILayout.BeginScrollView(_pfv3SelDropScroll, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(220f) });
		for (int i = 0; i < _pfv3Sel.Items.Count; i++)
		{
			PFV3Captured pFV3Captured = _pfv3Sel.Items[i];
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			if (GUILayout.Button((_pfv3SelFocus == i) ? ("★ " + pFV3Captured.Display) : pFV3Captured.Display, _sidebarBtn, Array.Empty<GUILayoutOption>()))
			{
				_pfv3SelFocus = i;
			}
			if (GUILayout.Button("✕", _btn, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(28f) }))
			{
				_pfv3Sel.Items.RemoveAt(i);
				if (_pfv3SelFocus >= _pfv3Sel.Items.Count)
				{
					_pfv3SelFocus = _pfv3Sel.Items.Count - 1;
				}
				i--;
			}
			GUILayout.EndHorizontal();
		}
		GUILayout.EndScrollView();
		GUILayout.EndArea();
	}

	private void EnsureEditorCameraActive(bool on)
	{
		//IL_01ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b6: Expected O, but got Unknown
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Expected O, but got Unknown
		//IL_01da: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e5: Expected O, but got Unknown
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_0027: Expected O, but got Unknown
		//IL_01f9: Unknown result type (might be due to invalid IL or missing references)
		//IL_0204: Expected O, but got Unknown
		//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f4: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0136: Expected O, but got Unknown
		//IL_0115: Unknown result type (might be due to invalid IL or missing references)
		//IL_011b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0149: Unknown result type (might be due to invalid IL or missing references)
		//IL_0154: Expected O, but got Unknown
		//IL_0188: Unknown result type (might be due to invalid IL or missing references)
		//IL_0193: Expected O, but got Unknown
		if (on)
		{
			if ((Object)_pfv3EditorCam == (Object)null)
			{
				GameObject val = new GameObject("PFV3_EditorCamera");
				_pfv3EditorCam = val.AddComponent<Camera>();
				_pfv3EditorCam.clearFlags = (CameraClearFlags)1;
				_pfv3EditorCam.nearClipPlane = 0.01f;
				_pfv3EditorCam.farClipPlane = 2000f;
				_pfv3EditorCtl = val.AddComponent<EditorFlyCam>();
				_pfv3EditorCtl.speed = 8f;
				_pfv3EditorCtl.sprint = 3f;
				_pfv3EditorCtl.lookSens = 0.12f;
				_pfv3EditorCtl.requireRightMouse = false;
			}
			PlayerController current = PlayerController.Current;
			Camera val2 = (((Object)(object)current != (Object)null) ? current.Camera : null) ?? Camera.main;
			Transform val3 = Actions.find_head();
			if ((Object)(object)val2 != (Object)null)
			{
				((Component)_pfv3EditorCam).transform.SetPositionAndRotation(((Component)val2).transform.position, ((Component)val2).transform.rotation);
			}
			else if ((Object)(object)val3 != (Object)null)
			{
				((Component)_pfv3EditorCam).transform.SetPositionAndRotation(val3.position, val3.rotation);
			}
			if ((Object)_pfv3PrevMainCam == (Object)null)
			{
				_pfv3PrevMainCam = Camera.main;
			}
			if ((Object)_pfv3PrevMainCam != (Object)null)
			{
				((Component)_pfv3PrevMainCam).tag = "Untagged";
			}
			((Component)_pfv3EditorCam).tag = "MainCamera";
			((Behaviour)_pfv3EditorCam).enabled = true;
			if ((Object)_pfv3EditorCtl != (Object)null)
			{
				((Behaviour)_pfv3EditorCtl).enabled = true;
			}
		}
		else
		{
			if ((Object)_pfv3EditorCam != (Object)null)
			{
				((Component)_pfv3EditorCam).tag = "Untagged";
				((Behaviour)_pfv3EditorCam).enabled = false;
			}
			if ((Object)_pfv3EditorCtl != (Object)null)
			{
				((Behaviour)_pfv3EditorCtl).enabled = false;
			}
			if ((Object)_pfv3PrevMainCam != (Object)null)
			{
				((Component)_pfv3PrevMainCam).tag = "MainCamera";
				_pfv3PrevMainCam = null;
			}
		}
	}

	private void PFV3_Begin3D()
	{
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		PFV3_EnsureLineMat();
		_pfv3LineMat.SetPass(0);
		GL.PushMatrix();
		GL.MultMatrix(Matrix4x4.identity);
	}

	private void PFV3_End3D()
	{
		GL.PopMatrix();
	}

	private GameObject PFV3_RayPickPrefab(out Vector3 pos, out Quaternion rot, out uint hash, out string name)
	{
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_0008: Unknown result type (might be due to invalid IL or missing references)
		//IL_000d: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ca: Unknown result type (might be due to invalid IL or missing references)
		pos = default(Vector3);
		rot = Quaternion.identity;
		hash = 0u;
		name = null;
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return null;
		}
		Vector2 val = PFV3_MouseScreen();
		RaycastHit val2 = default(RaycastHit);
		if (!Physics.Raycast(pFV3_Cam.ScreenPointToRay(new Vector3(val.x, val.y, 0f)), ref val2, 2000f, -1, (QueryTriggerInteraction)1))
		{
			return null;
		}
		GameObject val3 = (((Object)(object)((RaycastHit)(ref val2)).collider != (Object)null) ? ((Component)((RaycastHit)(ref val2)).collider).gameObject : null);
		NetworkPrefab val4 = (((Object)(object)val3 != (Object)null) ? (val3.GetComponent<NetworkPrefab>() ?? val3.GetComponentInParent<NetworkPrefab>()) : null);
		if ((Object)(object)val4 == (Object)null)
		{
			return null;
		}
		pos = ((RaycastHit)(ref val2)).point;
		rot = ((Component)val4).transform.rotation;
		hash = val4.Hash;
		name = ((Object)val3).name;
		return val3;
	}

	private static Vector3 PFV3_SnapAxis(Vector3 delta, int axis, float step)
	{
		//IL_0000: Unknown result type (might be due to invalid IL or missing references)
		//IL_0002: Unknown result type (might be due to invalid IL or missing references)
		//IL_0007: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_002b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0043: Unknown result type (might be due to invalid IL or missing references)
		Vector3 val = PFV3_AxisMask(delta, axis);
		switch (axis)
		{
		case 0:
			val.x = Mathf.Round(val.x / step) * step;
			break;
		case 1:
			val.y = Mathf.Round(val.y / step) * step;
			break;
		default:
			val.z = Mathf.Round(val.z / step) * step;
			break;
		}
		return val;
	}

	private static Vector3? PFV3_RayPlane(Vector3 ro, Vector3 rd, Vector3 p0, Vector3 n)
	{
		//IL_0000: Unknown result type (might be due to invalid IL or missing references)
		//IL_0001: Unknown result type (might be due to invalid IL or missing references)
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_0021: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0041: Unknown result type (might be due to invalid IL or missing references)
		//IL_0042: Unknown result type (might be due to invalid IL or missing references)
		//IL_0044: Unknown result type (might be due to invalid IL or missing references)
		//IL_0049: Unknown result type (might be due to invalid IL or missing references)
		float num = Vector3.Dot(rd, n);
		if (Mathf.Abs(num) < 1E-06f)
		{
			return null;
		}
		float num2 = Vector3.Dot(p0 - ro, n) / num;
		if (num2 < 0f)
		{
			return null;
		}
		return ro + rd * num2;
	}

	private static Vector2 PFV3_GuiToScreen(Vector2 gui)
	{
		//IL_0000: Unknown result type (might be due to invalid IL or missing references)
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0013: Unknown result type (might be due to invalid IL or missing references)
		return new Vector2(gui.x, (float)Screen.height - gui.y);
	}

	private float PFV3_AngleDeltaOnRing(Vector3 pivot, int axis, Vector2 guiStart, Vector2 guiNow)
	{
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0031: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_004c: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		//IL_005a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0069: Unknown result type (might be due to invalid IL or missing references)
		//IL_006e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0073: Unknown result type (might be due to invalid IL or missing references)
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		//IL_007d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0094: Unknown result type (might be due to invalid IL or missing references)
		//IL_0099: Unknown result type (might be due to invalid IL or missing references)
		//IL_009a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00cb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e9: Unknown result type (might be due to invalid IL or missing references)
		//IL_010c: Unknown result type (might be due to invalid IL or missing references)
		//IL_010e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0113: Unknown result type (might be due to invalid IL or missing references)
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return 0f;
		}
		Vector3 val = PFV3_AxisDir(axis);
		((Vector3)(ref val)).Normalize();
		Ray val2 = pFV3_Cam.ScreenPointToRay(new Vector3(PFV3_GuiToScreen(guiStart).x, PFV3_GuiToScreen(guiStart).y, 0f));
		Ray val3 = pFV3_Cam.ScreenPointToRay(new Vector3(PFV3_GuiToScreen(guiNow).x, PFV3_GuiToScreen(guiNow).y, 0f));
		Vector3? val4 = PFV3_RayPlane(((Ray)(ref val2)).origin, ((Ray)(ref val2)).direction, pivot, val);
		Vector3? val5 = PFV3_RayPlane(((Ray)(ref val3)).origin, ((Ray)(ref val3)).direction, pivot, val);
		if (!val4.HasValue || !val5.HasValue)
		{
			return 0f;
		}
		Vector3 val6 = val4.Value - pivot;
		Vector3 normalized = ((Vector3)(ref val6)).normalized;
		val6 = val5.Value - pivot;
		Vector3 normalized2 = ((Vector3)(ref val6)).normalized;
		float num = Mathf.Acos(Mathf.Clamp(Vector3.Dot(normalized, normalized2), -1f, 1f)) * 57.29578f;
		float num2 = Mathf.Sign(Vector3.Dot(Vector3.Cross(normalized, normalized2), val));
		return num * num2;
	}

	private void PFV3_DrawLineScreen(Vector3 aW, Vector3 bW, Color col, int thicknessPx = 2)
	{
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0016: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		//IL_001e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_0051: Unknown result type (might be due to invalid IL or missing references)
		//IL_0052: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_0089: Unknown result type (might be due to invalid IL or missing references)
		//IL_009a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ff: Unknown result type (might be due to invalid IL or missing references)
		//IL_0106: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)(object)PFV3_Cam == (Object)null)
		{
			return;
		}
		Vector2 val = PFV3_Project(aW);
		Vector2 val2 = PFV3_Project(bW);
		PFV3_EnsureLineMat();
		_pfv3LineMat.SetPass(0);
		GL.PushMatrix();
		GL.LoadPixelMatrix(0f, (float)Screen.width, (float)Screen.height, 0f);
		Vector2 val3 = val2 - val;
		if (((Vector2)(ref val3)).sqrMagnitude < 0.0001f)
		{
			GL.PopMatrix();
			return;
		}
		((Vector2)(ref val3)).Normalize();
		Vector2 val4 = default(Vector2);
		((Vector2)(ref val4))._002Ector(0f - val3.y, val3.x);
		GL.Begin(1);
		GL.Color(col);
		int num = Mathf.Max(1, thicknessPx / 2);
		for (int i = -num; i <= num; i++)
		{
			Vector2 val5 = val4 * (float)i;
			GL.Vertex3(val.x + val5.x, (float)Screen.height - val.y - val5.y, 0f);
			GL.Vertex3(val2.x + val5.x, (float)Screen.height - val2.y - val5.y, 0f);
		}
		GL.End();
		GL.PopMatrix();
	}

	private void PFV3_DrawRingScreen(Vector3 center, int axis, float radiusWorld, Color col, int thicknessPx = 2)
	{
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		//IL_0029: Unknown result type (might be due to invalid IL or missing references)
		//IL_0046: Unknown result type (might be due to invalid IL or missing references)
		//IL_003f: Unknown result type (might be due to invalid IL or missing references)
		//IL_004b: Unknown result type (might be due to invalid IL or missing references)
		//IL_004c: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_0058: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_005a: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0064: Unknown result type (might be due to invalid IL or missing references)
		//IL_0091: Unknown result type (might be due to invalid IL or missing references)
		//IL_0092: Unknown result type (might be due to invalid IL or missing references)
		//IL_009a: Unknown result type (might be due to invalid IL or missing references)
		//IL_009f: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e5: Unknown result type (might be due to invalid IL or missing references)
		if ((Object)(object)PFV3_Cam != (Object)null)
		{
			PFV3_EnsureLineMat();
			_pfv3LineMat.SetPass(0);
			Vector3 val = PFV3_AxisDir(axis);
			Vector3 val2 = ((Mathf.Abs(Vector3.Dot(val, Vector3.up)) > 0.95f) ? Vector3.right : Vector3.up);
			Vector3 val3 = Vector3.Normalize(Vector3.Cross(val, val2));
			Vector3 val4 = Vector3.Normalize(Vector3.Cross(val, val3));
			Vector3[] array = (Vector3[])(object)new Vector3[65];
			for (int i = 0; i <= 64; i++)
			{
				float num = (float)i / 64f * (float)Math.PI * 2f;
				array[i] = center + (val3 * Mathf.Cos(num) + val4 * Mathf.Sin(num)) * radiusWorld;
			}
			for (int j = 0; j < 64; j++)
			{
				PFV3_DrawLineScreen(array[j], array[j + 1], col, thicknessPx);
			}
		}
	}

	private int PFV3_PickAxisHandle(Vector3 pivot)
	{
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_000b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_0047: Unknown result type (might be due to invalid IL or missing references)
		//IL_004d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0052: Unknown result type (might be due to invalid IL or missing references)
		//IL_0059: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0158: Unknown result type (might be due to invalid IL or missing references)
		//IL_015d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0162: Unknown result type (might be due to invalid IL or missing references)
		//IL_0179: Unknown result type (might be due to invalid IL or missing references)
		//IL_0184: Unknown result type (might be due to invalid IL or missing references)
		//IL_018b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0190: Unknown result type (might be due to invalid IL or missing references)
		//IL_0195: Unknown result type (might be due to invalid IL or missing references)
		//IL_019a: Unknown result type (might be due to invalid IL or missing references)
		//IL_019b: Unknown result type (might be due to invalid IL or missing references)
		//IL_01a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_01ac: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b7: Unknown result type (might be due to invalid IL or missing references)
		//IL_011c: Unknown result type (might be due to invalid IL or missing references)
		//IL_012b: Unknown result type (might be due to invalid IL or missing references)
		//IL_013a: Unknown result type (might be due to invalid IL or missing references)
		//IL_007b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0086: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0092: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d2: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d7: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_01dd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00af: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b5: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ba: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Unknown result type (might be due to invalid IL or missing references)
		Vector2 mp = PFV3_MouseScreen();
		float num = 256f;
		if (_pfv3Mode == PFV3Mode.Translate)
		{
			float num2 = 1.45f * _pfv3GizmoSize;
			int result = -1;
			float num3 = float.PositiveInfinity;
			float num4 = Seg(pivot, pivot + Vector3.right * num2);
			if (num4 < num3)
			{
				num3 = num4;
				result = 0;
			}
			float num5 = Seg(pivot, pivot + Vector3.up * num2);
			if (num5 < num3)
			{
				num3 = num5;
				result = 1;
			}
			float num6 = Seg(pivot, pivot + Vector3.forward * num2);
			if (num6 < num3)
			{
				num3 = num6;
				result = 2;
			}
			if (!(num3 <= num))
			{
				return -1;
			}
			return result;
		}
		int best;
		float bestErr;
		float R;
		if (_pfv3Mode == PFV3Mode.Rotate)
		{
			best = -1;
			bestErr = 16f;
			R = 1.2f * _pfv3GizmoSize;
			Test(Vector3.right, 0);
			Test(Vector3.up, 1);
			Test(Vector3.forward, 2);
			return best;
		}
		Camera pFV3_Cam = PFV3_Cam;
		Vector2 val = PFV3_Project(pivot);
		float num7 = 1.2f * _pfv3GizmoSize * 0.95f;
		Vector2 val2 = PFV3_Project(pivot + ((Component)pFV3_Cam).transform.right * num7) - val;
		float magnitude = ((Vector2)(ref val2)).magnitude;
		val2 = mp - val;
		if (Mathf.Abs(((Vector2)(ref val2)).magnitude - magnitude) <= 16f)
		{
			return 3;
		}
		val2 = mp - val;
		if (((Vector2)(ref val2)).sqrMagnitude <= num * 0.36f)
		{
			return 3;
		}
		return -1;
		float Seg(Vector3 a, Vector3 b)
		{
			//IL_0001: Unknown result type (might be due to invalid IL or missing references)
			//IL_0002: Unknown result type (might be due to invalid IL or missing references)
			//IL_0007: Unknown result type (might be due to invalid IL or missing references)
			//IL_0009: Unknown result type (might be due to invalid IL or missing references)
			//IL_000a: Unknown result type (might be due to invalid IL or missing references)
			//IL_000f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0011: Unknown result type (might be due to invalid IL or missing references)
			//IL_0016: Unknown result type (might be due to invalid IL or missing references)
			//IL_0017: Unknown result type (might be due to invalid IL or missing references)
			Vector2 a2 = PFV3_Project(a);
			Vector2 b2 = PFV3_Project(b);
			return PFV3_SqrDistPointSegment(mp, a2, b2);
		}
		bool Test(Vector3 normal, int axis)
		{
			//IL_0009: Unknown result type (might be due to invalid IL or missing references)
			//IL_000e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0013: Unknown result type (might be due to invalid IL or missing references)
			//IL_0014: Unknown result type (might be due to invalid IL or missing references)
			//IL_001b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0020: Unknown result type (might be due to invalid IL or missing references)
			//IL_0025: Unknown result type (might be due to invalid IL or missing references)
			//IL_0049: Unknown result type (might be due to invalid IL or missing references)
			//IL_004e: Unknown result type (might be due to invalid IL or missing references)
			//IL_0055: Unknown result type (might be due to invalid IL or missing references)
			//IL_005a: Unknown result type (might be due to invalid IL or missing references)
			//IL_005f: Unknown result type (might be due to invalid IL or missing references)
			//IL_0064: Unknown result type (might be due to invalid IL or missing references)
			//IL_0065: Unknown result type (might be due to invalid IL or missing references)
			//IL_006a: Unknown result type (might be due to invalid IL or missing references)
			//IL_0075: Unknown result type (might be due to invalid IL or missing references)
			//IL_007a: Unknown result type (might be due to invalid IL or missing references)
			//IL_007b: Unknown result type (might be due to invalid IL or missing references)
			//IL_0080: Unknown result type (might be due to invalid IL or missing references)
			//IL_0034: Unknown result type (might be due to invalid IL or missing references)
			//IL_0035: Unknown result type (might be due to invalid IL or missing references)
			//IL_003a: Unknown result type (might be due to invalid IL or missing references)
			//IL_003f: Unknown result type (might be due to invalid IL or missing references)
			Camera pFV3_Cam2 = PFV3_Cam;
			Vector2 val3 = PFV3_Project(pivot);
			Vector3 val4 = Vector3.Cross(normal, ((Component)pFV3_Cam2).transform.forward);
			if (((Vector3)(ref val4)).sqrMagnitude < 1E-06f)
			{
				val4 = Vector3.Cross(normal, Vector3.right);
			}
			((Vector3)(ref val4)).Normalize();
			Vector2 val5 = PFV3_Project(pivot + val4 * R) - val3;
			float magnitude2 = ((Vector2)(ref val5)).magnitude;
			val5 = mp - val3;
			float num8 = Mathf.Abs(((Vector2)(ref val5)).magnitude - magnitude2);
			if (num8 <= 16f && num8 < bestErr)
			{
				bestErr = num8;
				best = axis;
				return true;
			}
			return false;
		}
	}

	private static string PFV3_Stringify(float value)
	{
		return BitConverter.ToUInt32(BitConverter.GetBytes(value), 0).ToString();
	}

	private static string PFV3_BuildSpawnCsv(uint hash, Vector3 pos, Quaternion rot, float uniformScale, bool kinematic)
	{
		StringBuilder stringBuilder = new StringBuilder(160);
		stringBuilder.Append(hash).Append(",45,").Append(hash)
			.Append(",");
		stringBuilder.Append(pos.x.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(pos.y.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(pos.z.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(rot.x.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(rot.y.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(rot.z.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(rot.w.ToString("0.########", Inv)).Append(",");
		stringBuilder.Append(Mathf.Max(0.0001f, uniformScale).ToString("0.######", Inv)).Append(",");
		stringBuilder.Append(kinematic ? "1" : "0").Append(",0,0,");
		return stringBuilder.ToString();
	}

	private static Vector3 PFV3_AxisMask(Vector3 v, int axis)
	{
		//IL_000a: Unknown result type (might be due to invalid IL or missing references)
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		return new Vector3((axis == 0) ? v.x : 0f, (axis == 1) ? v.y : 0f, (axis == 2) ? v.z : 0f);
	}

	private Vector3 PFV3_ComputePivot()
	{
		//IL_0018: Unknown result type (might be due to invalid IL or missing references)
		//IL_001d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0039: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_005f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3Sel.Items.Count == 0)
		{
			return Vector3.zero;
		}
		Vector3 val = Vector3.zero;
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			val += item.Pos;
		}
		return val / (float)_pfv3Sel.Items.Count;
	}

	private void PFV3_ConfirmToServer()
	{
		//IL_0065: Unknown result type (might be due to invalid IL or missing references)
		//IL_006a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		//IL_0076: Unknown result type (might be due to invalid IL or missing references)
		//IL_007d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0082: Unknown result type (might be due to invalid IL or missing references)
		//IL_0118: Unknown result type (might be due to invalid IL or missing references)
		//IL_011f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0162: Unknown result type (might be due to invalid IL or missing references)
		//IL_0167: Unknown result type (might be due to invalid IL or missing references)
		//IL_016d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0172: Unknown result type (might be due to invalid IL or missing references)
		//IL_0178: Unknown result type (might be due to invalid IL or missing references)
		//IL_017d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0184: Unknown result type (might be due to invalid IL or missing references)
		//IL_0189: Unknown result type (might be due to invalid IL or missing references)
		if (_pfv3Sel.Items.Count == 0)
		{
			return;
		}
		PFV3_ApplyPreviewLocally();
		foreach (PFV3Captured item in _pfv3Sel.Items)
		{
			Transform val = (((Object)(object)item.Tr != (Object)null) ? item.Tr : (item.Tr = PFV3_FindTransformFor(item)));
			if ((Object)(object)val != (Object)null)
			{
				item.Pos = val.position;
				item.Rot = val.rotation;
				item.Scale = val.localScale;
			}
		}
		if (_pfv3Tool != PFV3Tool.Clone)
		{
			PFV3_DeleteSelected();
		}
		foreach (PFV3Captured item2 in _pfv3Sel.Items)
		{
			if (item2.Hash != 0)
			{
				float uniformScale = Mathf.Max(0.0001f, (item2.Scale.x + item2.Scale.y + item2.Scale.z) / 3f);
				string text = PFV3_BuildSpawnCsv(item2.Hash, item2.Pos, item2.Rot, uniformScale, _pfv3Kinematic);
				SendConsoleCmd("spawn string-raw " + text);
			}
		}
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3PreviewScale = Vector3.one;
		_pfv3Pivot = PFV3_ComputePivot();
		_pfv3Sel.Phase = PFV3SelPhase.Selected;
	}

	private void PFV3_Undo()
	{
		if (_pfv3Undo.Count != 0)
		{
			PFV3Snapshot item = PFV3_CaptureSnapshot();
			PFV3Snapshot snap = _pfv3Undo.Pop();
			_pfv3Redo.Push(item);
			PFV3_ApplySnapshot(snap);
			_statusLeft = "Undo (local).";
		}
	}

	private void PFV3_Redo()
	{
		if (_pfv3Redo.Count != 0)
		{
			PFV3Snapshot item = PFV3_CaptureSnapshot();
			PFV3Snapshot snap = _pfv3Redo.Pop();
			_pfv3Undo.Push(item);
			PFV3_ApplySnapshot(snap);
			_statusLeft = "Redo (local).";
		}
	}

	private void PFV3_EraseServer()
	{
		if (!_pfv3Sel.Active || _pfv3Sel.Items.Count == 0)
		{
			return;
		}
		foreach (IGrouping<uint, PFV3Captured> item in from i in _pfv3Sel.Items
			group i by i.Hash)
		{
			SendConsoleCmd($"select {item.Key}");
			SendConsoleCmd("select destroy");
		}
		_statusLeft = "Erase: requested destroy for grouped selection.";
	}

	private void PFV3_Reset()
	{
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Unknown result type (might be due to invalid IL or missing references)
		//IL_001c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0030: Unknown result type (might be due to invalid IL or missing references)
		//IL_0035: Unknown result type (might be due to invalid IL or missing references)
		_pfv3Sel.Clear();
		_pfv3PreviewOffset = Vector3.zero;
		_pfv3PreviewEuler = Vector3.zero;
		_pfv3Dragging = false;
		_pfv3ActiveAxis = -1;
		_pfv3PreviewScale = Vector3.one;
		_pfv3Undo.Clear();
		_pfv3Redo.Clear();
	}

	private Vector3? PFV3_ClosestPointOnAxis(Vector3 pivot, int axis)
	{
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0020: Unknown result type (might be due to invalid IL or missing references)
		//IL_0022: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		//IL_0033: Unknown result type (might be due to invalid IL or missing references)
		//IL_0038: Unknown result type (might be due to invalid IL or missing references)
		//IL_003d: Unknown result type (might be due to invalid IL or missing references)
		//IL_004e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0053: Unknown result type (might be due to invalid IL or missing references)
		//IL_005e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0060: Unknown result type (might be due to invalid IL or missing references)
		//IL_0063: Unknown result type (might be due to invalid IL or missing references)
		//IL_0068: Unknown result type (might be due to invalid IL or missing references)
		//IL_006c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0071: Unknown result type (might be due to invalid IL or missing references)
		//IL_0075: Unknown result type (might be due to invalid IL or missing references)
		//IL_007c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0081: Unknown result type (might be due to invalid IL or missing references)
		//IL_0083: Unknown result type (might be due to invalid IL or missing references)
		//IL_0084: Unknown result type (might be due to invalid IL or missing references)
		//IL_008c: Unknown result type (might be due to invalid IL or missing references)
		//IL_008d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0096: Unknown result type (might be due to invalid IL or missing references)
		//IL_0098: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a3: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ab: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b4: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b6: Unknown result type (might be due to invalid IL or missing references)
		//IL_0057: Unknown result type (might be due to invalid IL or missing references)
		//IL_005c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f2: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00da: Unknown result type (might be due to invalid IL or missing references)
		Camera pFV3_Cam = PFV3_Cam;
		if ((Object)(object)pFV3_Cam == (Object)null)
		{
			return null;
		}
		Vector2 val = PFV3_MouseScreen();
		Ray val2 = pFV3_Cam.ScreenPointToRay(new Vector3(val.x, val.y, 0f));
		Vector3 val3 = (Vector3)(axis switch
		{
			1 => Vector3.up, 
			0 => Vector3.right, 
			_ => Vector3.forward, 
		});
		Vector3 origin = ((Ray)(ref val2)).origin;
		Vector3 direction = ((Ray)(ref val2)).direction;
		Vector3 normalized = ((Vector3)(ref direction)).normalized;
		Vector3 normalized2 = ((Vector3)(ref val3)).normalized;
		float num = Vector3.Dot(normalized, normalized);
		float num2 = Vector3.Dot(normalized, normalized2);
		float num3 = Vector3.Dot(normalized2, normalized2);
		Vector3 val4 = origin - pivot;
		float num4 = Vector3.Dot(normalized, val4);
		float num5 = Vector3.Dot(normalized2, val4);
		float num6 = num * num3 - num2 * num2;
		if (Mathf.Abs(num6) < 1E-05f)
		{
			return pivot;
		}
		float num7 = (num2 * num4 - num * num5) / num6;
		return pivot + normalized2 * num7;
	}

	private void DrawUtilitiesTab()
	{
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0025: Expected O, but got Unknown
		//IL_011f: Unknown result type (might be due to invalid IL or missing references)
		//IL_012a: Expected O, but got Unknown
		//IL_0133: Unknown result type (might be due to invalid IL or missing references)
		//IL_0138: Unknown result type (might be due to invalid IL or missing references)
		//IL_01bc: Unknown result type (might be due to invalid IL or missing references)
		//IL_01c7: Expected O, but got Unknown
		//IL_01da: Unknown result type (might be due to invalid IL or missing references)
		//IL_0160: Unknown result type (might be due to invalid IL or missing references)
		//IL_0222: Unknown result type (might be due to invalid IL or missing references)
		//IL_022d: Expected O, but got Unknown
		//IL_0231: Unknown result type (might be due to invalid IL or missing references)
		//IL_023c: Expected O, but got Unknown
		//IL_0247: Unknown result type (might be due to invalid IL or missing references)
		//IL_024c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0256: Unknown result type (might be due to invalid IL or missing references)
		//IL_025b: Unknown result type (might be due to invalid IL or missing references)
		GUILayout.Label("Utilities", _h1, Array.Empty<GUILayoutOption>());
		if ((Object)PlayerController.Current != (Object)null && ((Object)PlayerController.Current).name == "Customization Controller(Clone)" && GUILayout.Button("Escape Customization", _btn, Array.Empty<GUILayoutOption>()))
		{
			try
			{
				Actions.escape_customization();
			}
			catch (Exception ex)
			{
				MelonLogger.Warning("escape_customization failed: " + ex.Message);
			}
		}
		if (GUILayout.Button("Toggle Enemy Health Bars", _btn, Array.Empty<GUILayoutOption>()))
		{
			ToggleEnemyHealthBars val = ScriptableObject.CreateInstance<ToggleEnemyHealthBars>();
			if ((Object)(object)val != (Object)null)
			{
				((QuickAccessMenuAction)val).Run((Controller)null);
			}
		}
		if (GUILayout.Button("Equipment Mode", _btn, Array.Empty<GUILayoutOption>()))
		{
			EquipmentModeQuickAccess val2 = ScriptableObject.CreateInstance<EquipmentModeQuickAccess>();
			if ((Object)(object)val2 != (Object)null)
			{
				((QuickAccessMenuAction)val2).Run((Controller)null);
			}
		}
		if (GUILayout.Button("Summon Camera", _btn, Array.Empty<GUILayoutOption>()))
		{
			SummonCameraQuickAccess val3 = ScriptableObject.CreateInstance<SummonCameraQuickAccess>();
			if ((Object)(object)val3 != (Object)null)
			{
				((QuickAccessMenuAction)val3).Run((Controller)null);
			}
		}
		if (GUILayout.Button("Toggle Terrain", _btn, Array.Empty<GUILayoutOption>()))
		{
			GameObject val4 = GameObject.Find("MasterTerrain");
			if ((Object)val4 != (Object)null)
			{
				Vector3 position = val4.transform.position;
				position.y += (_terrainLowered ? 20f : (-20f));
				val4.transform.position = position;
				_terrainLowered = !_terrainLowered;
			}
		}
		if (GUILayout.Button("Bag Looking", _btn, Array.Empty<GUILayoutOption>()))
		{
			ToggleStorage();
		}
		GUILayout.Space(8f);
		if (GUILayout.Button("Teleport Player → Camera", _btn, Array.Empty<GUILayoutOption>()))
		{
			Camera main = Camera.main;
			if ((Object)main != (Object)null)
			{
				PlayerController.Current.LocomotionController.MoveTo(((Component)main).transform.position, (LocomotionFunction)0);
			}
		}
		if (GUILayout.Button("Teleport Camera → Player", _btn, Array.Empty<GUILayoutOption>()))
		{
			PlayerController current = PlayerController.Current;
			Camera main2 = Camera.main;
			Transform val5 = (((Object)(object)main2 != (Object)null) ? ((Component)main2).transform : null);
			if ((Object)current != (Object)null && (Object)val5 != (Object)null)
			{
				val5.position = ((Component)current).transform.position + Vector3.up * 1.6f;
			}
		}
	}

	private static string ExtractRawToken(string tokenOrComposed)
	{
		if (string.IsNullOrWhiteSpace(tokenOrComposed))
		{
			return "";
		}
		string text = tokenOrComposed.Trim();
		int num = text.IndexOf('|');
		if (num < 0)
		{
			return text;
		}
		return text.Substring(0, num);
	}

	private static bool LooksLikeToken(string s)
	{
		if (string.IsNullOrWhiteSpace(s))
		{
			return false;
		}
		s = s.Trim();
		if (s.StartsWith("TOKEN|", StringComparison.OrdinalIgnoreCase))
		{
			return true;
		}
		if (s.Contains("|"))
		{
			return true;
		}
		if (s.Length >= 32 && s.All((char ch) => char.IsLetterOrDigit(ch) || ch == '-' || ch == '_' || ch == '.'))
		{
			return true;
		}
		return false;
	}

	private void EnqueueChat(string sender, string text, Color color)
	{
		//IL_0019: Unknown result type (might be due to invalid IL or missing references)
		lock (_pendingLock)
		{
			_pendingChat.Enqueue(new ChatMessage(sender, text, color));
		}
	}

	private async Task EnsureWsConnectedAsync()
	{
		if (_ws != null && _ws.State == WebSocketState.Open)
		{
			return;
		}
		try
		{
			_wsCts?.Cancel();
			_ws?.Dispose();
			_ws = new ClientWebSocket();
			_wsCts = new CancellationTokenSource();
			if (!string.IsNullOrWhiteSpace(_save?.ApiToken))
			{
				_ws.Options.SetRequestHeader("Authorization", _save.ApiToken.Trim());
			}
			Uri uri = new Uri("wss://romme.potatobot.org/ws");
			MelonLogger.Msg($"[WS] Connecting to {uri} …");
			await _ws.ConnectAsync(uri, _wsCts.Token);
			MelonLogger.Msg("[WS] Connected.");
			StartWsReceiveLoop();
		}
		catch (Exception ex)
		{
			MelonLogger.Warning("[WS] Connect failed: " + ex.GetBaseException().Message);
			throw;
		}
	}

	private async Task WsSendAsync(string text)
	{
		if (_ws == null || _ws.State != WebSocketState.Open)
		{
			await EnsureWsConnectedAsync();
		}
		byte[] bytes = Encoding.UTF8.GetBytes(text);
		await _ws.SendAsync(new ArraySegment<byte>(bytes), WebSocketMessageType.Text, endOfMessage: true, _wsCts?.Token ?? CancellationToken.None);
	}

	private void StartWsReceiveLoop()
	{
		_wsRecvTask = Task.Run(async delegate
		{
			byte[] buffer = new byte[4096];
			try
			{
				while (_ws != null && _ws.State == WebSocketState.Open)
				{
					WebSocketReceiveResult webSocketReceiveResult = await _ws.ReceiveAsync(new ArraySegment<byte>(buffer), _wsCts.Token);
					if (webSocketReceiveResult.MessageType == WebSocketMessageType.Close)
					{
						MelonLogger.Msg($"[WS] Closed: {_ws.CloseStatus} {_ws.CloseStatusDescription}");
						break;
					}
					string text = Encoding.UTF8.GetString(buffer, 0, webSocketReceiveResult.Count);
					if (!string.IsNullOrWhiteSpace(text))
					{
						if (text.StartsWith("sessionid"))
						{
							_wsAlive = true;
						}
						else if (!(text == "ping") && !text.Contains("\"type\":\"ping\"") && !text.Contains("\"type\":\"ping_event\"") && !text.Contains("\"type\":\"login_response\""))
						{
							try
							{
								JObject val = JObject.Parse(text);
								switch (((object)val["type"])?.ToString())
								{
								case "incoming_message":
								{
									JToken val3 = val["data"];
									object obj;
									if (val3 == null)
									{
										obj = null;
									}
									else
									{
										JToken val4 = val3[(object)"sender"];
										obj = ((val4 == null) ? null : ((object)val4[(object)"username"])?.ToString());
									}
									if (obj == null)
									{
										obj = "server";
									}
									string sender = (string)obj;
									string text3 = ((val3 == null) ? null : ((object)val3[(object)"message"])?.ToString()) ?? "";
									EnqueueChat(sender, text3, Color.white);
									break;
								}
								case "user_joined_server_event":
								{
									JToken val5 = val["data"];
									object obj2;
									if (val5 == null)
									{
										obj2 = null;
									}
									else
									{
										JToken val6 = val5[(object)"user"];
										if (val6 == null)
										{
											obj2 = null;
										}
										else
										{
											JToken val7 = val6[(object)"user"];
											obj2 = ((val7 == null) ? null : ((object)val7[(object)"username"])?.ToString());
										}
									}
									if (obj2 == null)
									{
										obj2 = "someone";
									}
									string text4 = (string)obj2;
									EnqueueChat("network", text4 + " has joined your server", Color.cyan);
									break;
								}
								case "command_response":
								{
									JToken val2 = val["data"];
									string text2 = ((val2 == null) ? null : ((object)val2[(object)"message"])?.ToString()) ?? ((val2 == null) ? null : ((object)val2[(object)"output"])?.ToString()) ?? ((val2 == null) ? null : ((object)val2[(object)"response"])?.ToString()) ?? ((val2 == null) ? null : ((object)val2[(object)"result"])?.ToString()) ?? ((val2 == null) ? null : ((object)val2[(object)"words"])?.ToString()) ?? ((object)val2)?.ToString() ?? "";
									EnqueueChat("", text2, Color.white);
									break;
								}
								default:
									EnqueueChat("", text, Color.white);
									break;
								}
							}
							catch
							{
								EnqueueChat("", text, Color.white);
							}
						}
					}
				}
			}
			catch (Exception ex)
			{
				MelonLogger.Warning("[WS] Receive error: " + ex.GetBaseException().Message);
			}
			finally
			{
				_wsAuthenticated = false;
			}
		});
	}

	private void HookApiEvents()
	{
		if (_api != null)
		{
			try
			{
				_api.ChatReceived -= OnApiChatReceived;
			}
			catch
			{
			}
			try
			{
				_api.SystemMessage -= OnApiSystem;
			}
			catch
			{
			}
			try
			{
				_api.Warning -= OnApiWarn;
			}
			catch
			{
			}
			try
			{
				_api.Disconnected -= OnApiDisconnected;
			}
			catch
			{
			}
			_api.ChatReceived += OnApiChatReceived;
			_api.SystemMessage += OnApiSystem;
			_api.Warning += OnApiWarn;
			_api.Disconnected += OnApiDisconnected;
		}
	}

	private void EnsureChatStyles()
	{
		//IL_002d: Unknown result type (might be due to invalid IL or missing references)
		//IL_0032: Unknown result type (might be due to invalid IL or missing references)
		//IL_003b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0045: Expected O, but got Unknown
		//IL_0045: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0054: Expected O, but got Unknown
		//IL_0055: Expected O, but got Unknown
		//IL_0072: Unknown result type (might be due to invalid IL or missing references)
		//IL_0093: Unknown result type (might be due to invalid IL or missing references)
		//IL_0098: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00b3: Expected O, but got Unknown
		//IL_00b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_00be: Unknown result type (might be due to invalid IL or missing references)
		//IL_00c6: Expected O, but got Unknown
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fc: Unknown result type (might be due to invalid IL or missing references)
		//IL_0101: Unknown result type (might be due to invalid IL or missing references)
		//IL_0109: Unknown result type (might be due to invalid IL or missing references)
		//IL_010e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0118: Expected O, but got Unknown
		//IL_011d: Expected O, but got Unknown
		if (_chatPanelStyle == null || _chatStyle == null || _chatSenderStyle == null || _chatInputStyle == null)
		{
			GUIStyle val = new GUIStyle(GUI.skin.box)
			{
				padding = new RectOffset(10, 10, 10, 10),
				margin = new RectOffset(0, 0, 0, 0)
			};
			val.normal.background = MakeTex(2, 2, new Color(0f, 0f, 0f, 0.62f));
			_chatPanelStyle = val;
			_chatStyle = new GUIStyle(GUI.skin.label)
			{
				fontSize = 16,
				wordWrap = true,
				richText = true
			};
			GUIStyle val2 = new GUIStyle(_chatStyle)
			{
				fontStyle = (FontStyle)1
			};
			val2.normal.textColor = new Color(0.7f, 0.85f, 1f, 1f);
			_chatSenderStyle = val2;
			_chatInputStyle = new GUIStyle(GUI.skin.textField)
			{
				fontSize = 16,
				padding = new RectOffset(6, 6, 6, 6)
			};
		}
	}

	private void OnApiChatReceived(string sender, string text, Color color)
	{
		//IL_0003: Unknown result type (might be due to invalid IL or missing references)
		EnqueueChat(sender, text, color);
	}

	private void OnApiSystem(string msg)
	{
		//IL_0010: Unknown result type (might be due to invalid IL or missing references)
		EnqueueChat("system", msg ?? "", Color.yellow);
	}

	private void OnApiWarn(string msg)
	{
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		EnqueueChat("warn", msg ?? "", new Color(1f, 0.6f, 0.2f));
	}

	private void OnApiDisconnected()
	{
		ResetApiSessionState();
	}

	private void DrawChatOverlay()
	{
		//IL_00db: Unknown result type (might be due to invalid IL or missing references)
		//IL_00e0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00eb: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f0: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f6: Unknown result type (might be due to invalid IL or missing references)
		//IL_00f8: Unknown result type (might be due to invalid IL or missing references)
		//IL_00fe: Unknown result type (might be due to invalid IL or missing references)
		//IL_0104: Unknown result type (might be due to invalid IL or missing references)
		//IL_0110: Unknown result type (might be due to invalid IL or missing references)
		//IL_011a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0121: Unknown result type (might be due to invalid IL or missing references)
		//IL_0128: Unknown result type (might be due to invalid IL or missing references)
		//IL_0135: Unknown result type (might be due to invalid IL or missing references)
		//IL_0140: Unknown result type (might be due to invalid IL or missing references)
		//IL_0161: Unknown result type (might be due to invalid IL or missing references)
		//IL_018a: Unknown result type (might be due to invalid IL or missing references)
		//IL_018f: Unknown result type (might be due to invalid IL or missing references)
		//IL_01b9: Unknown result type (might be due to invalid IL or missing references)
		//IL_01d8: Unknown result type (might be due to invalid IL or missing references)
		//IL_0271: Unknown result type (might be due to invalid IL or missing references)
		//IL_0292: Unknown result type (might be due to invalid IL or missing references)
		//IL_035b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0361: Unknown result type (might be due to invalid IL or missing references)
		if (!_chatOverlayVisible && !_chatTyping && _chatFadeAlpha <= 0.01f)
		{
			return;
		}
		float realtimeSinceStartup = Time.realtimeSinceStartup;
		if (_chatTyping)
		{
			TouchChatActivity();
		}
		else
		{
			float num = realtimeSinceStartup - _chatLastActivity;
			if (num > 6f)
			{
				float num2 = (num - 6f) / 2.5f;
				_chatFadeAlpha = Mathf.Clamp01(1f - num2);
			}
			else
			{
				_chatFadeAlpha = 1f;
			}
			if (_chatFadeAlpha <= 0.01f && !_chatTyping)
			{
				return;
			}
		}
		float num3 = Mathf.Clamp((float)Screen.width * 0.38f, 360f, 520f);
		float num4 = Mathf.Clamp((float)Screen.height * 0.42f, 260f, 440f);
		_chatRect = new Rect(18f, (float)Screen.height - num4 - 32f, num3, num4);
		EnsureChatStyles();
		Color color = GUI.color;
		Color contentColor = GUI.contentColor;
		GUI.color = new Color(color.r, color.g, color.b, _chatFadeAlpha);
		GUI.contentColor = new Color(contentColor.r, contentColor.g, contentColor.b, _chatFadeAlpha);
		GUILayout.BeginArea(_chatRect, GUIContent.none, _chatPanelStyle);
		GUILayout.BeginVertical(Array.Empty<GUILayoutOption>());
		_chatScroll = GUILayout.BeginScrollView(_chatScroll, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(((Rect)(ref _chatRect)).height - 44f) });
		int count = _chat.Count;
		if (count == 0)
		{
			GUI.contentColor = new Color(1f, 1f, 1f, 0.5f);
			GUILayout.Label("No messages yet.", _chatStyle, Array.Empty<GUILayoutOption>());
			GUI.contentColor = contentColor;
		}
		else
		{
			for (int num5 = Mathf.Min(count, 150) - 1; num5 >= 0; num5--)
			{
				ChatMessage chatMessage = _chat[num5];
				GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
				if (!string.IsNullOrEmpty(chatMessage.Sender))
				{
					GUILayout.Label("[" + chatMessage.Sender + "]", _chatSenderStyle, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Width(Mathf.Min(140f, ((Rect)(ref _chatRect)).width * 0.35f)) });
				}
				GUI.contentColor = chatMessage.Color;
				GUILayout.Label(chatMessage.Text, _chatStyle, Array.Empty<GUILayoutOption>());
				GUI.contentColor = contentColor;
				GUILayout.EndHorizontal();
			}
		}
		GUILayout.EndScrollView();
		if (_chatNewMessage)
		{
			_chatScroll.y = float.MaxValue;
			_chatNewMessage = false;
		}
		GUILayout.Space(4f);
		if (_chatTyping)
		{
			ChatTypingLogic();
		}
		else if (_chatOverlayVisible)
		{
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.FlexibleSpace();
			if (GUILayout.Button("Type (T)", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
			{
				GUILayout.Width(90f),
				GUILayout.Height(24f)
			}))
			{
				_chatTyping = true;
				_chatFocusPending = true;
				TouchChatActivity();
			}
			GUILayout.EndHorizontal();
		}
		GUILayout.EndVertical();
		GUILayout.EndArea();
		GUI.color = color;
		GUI.contentColor = contentColor;
	}

	private void DrawTokenPopup()
	{
		//IL_003b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0103: Unknown result type (might be due to invalid IL or missing references)
		//IL_0109: Invalid comparison between Unknown and I4
		//IL_010c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0113: Invalid comparison between Unknown and I4
		//IL_0116: Unknown result type (might be due to invalid IL or missing references)
		//IL_0120: Invalid comparison between Unknown and I4
		if (_awaitingToken)
		{
			EnsureChatStyles();
			float num = 420f;
			float num2 = 200f;
			Rect val = new Rect(((float)Screen.width - num) * 0.5f, ((float)Screen.height - num2) * 0.35f, num, num2);
			int depth = GUI.depth;
			GUI.depth = -100;
			GUILayout.BeginArea(val, GUIContent.none, _chatPanelStyle);
			GUILayout.BeginVertical(Array.Empty<GUILayoutOption>());
			GUILayout.Label("API Token Required", _h2, Array.Empty<GUILayoutOption>());
			GUILayout.Label("Paste your Basemodal API token below to enable chat and features.", _small, Array.Empty<GUILayoutOption>());
			GUILayout.Space(6f);
			GUI.SetNextControlName("TokenField");
			_tokenInput = GUILayout.TextField(_tokenInput ?? "", _chatInputStyle, (GUILayoutOption[])(object)new GUILayoutOption[1] { GUILayout.Height(28f) });
			if (_tokenFocusPending)
			{
				GUI.FocusControl("TokenField");
				_tokenFocusPending = false;
			}
			Event current = Event.current;
			bool flag = false;
			if (current != null && (int)current.type == 4 && ((int)current.keyCode == 13 || (int)current.keyCode == 271))
			{
				flag = true;
				current.Use();
			}
			GUILayout.Space(8f);
			GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
			GUILayout.FlexibleSpace();
			GUI.enabled = !_tokenSubmitting;
			if (GUILayout.Button(_tokenSubmitting ? "Connecting..." : "Save & Connect", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
			{
				GUILayout.Width(160f),
				GUILayout.Height(28f)
			}))
			{
				flag = true;
			}
			GUI.enabled = true;
			GUILayout.EndHorizontal();
			if (flag && !_tokenSubmitting)
			{
				SetupApiTokenAsync(_tokenInput);
			}
			GUILayout.EndVertical();
			GUILayout.EndArea();
			GUI.depth = depth;
		}
	}

	private void RequestChatFocus()
	{
		_chatFocusPending = true;
		TouchChatActivity();
	}

	private void AppendChatMessage(ChatMessage msg)
	{
		if (msg != null)
		{
			_chat.Insert(0, msg);
			if (_chat.Count > 150)
			{
				_chat.RemoveRange(150, _chat.Count - 150);
			}
			_chatNewMessage = true;
			TouchChatActivity();
		}
	}

	private void AddChatMessage(string sender, string text, Color? color = null)
	{
		//IL_0023: Unknown result type (might be due to invalid IL or missing references)
		//IL_001a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0028: Unknown result type (might be due to invalid IL or missing references)
		//IL_002c: Unknown result type (might be due to invalid IL or missing references)
		Color color2 = (Color)(((_003F?)color) ?? new Color(1f, 0.5f, 0f));
		AppendChatMessage(new ChatMessage(sender, text, color2));
	}

	private void TouchChatActivity()
	{
		_chatOverlayVisible = true;
		_chatFadeAlpha = 1f;
		_chatLastActivity = Time.realtimeSinceStartup;
	}

	private void ChatTypingLogic()
	{
		//IL_009e: Unknown result type (might be due to invalid IL or missing references)
		//IL_00a4: Invalid comparison between Unknown and I4
		//IL_00a7: Unknown result type (might be due to invalid IL or missing references)
		//IL_00ae: Invalid comparison between Unknown and I4
		//IL_00b1: Unknown result type (might be due to invalid IL or missing references)
		//IL_00bb: Invalid comparison between Unknown and I4
		//IL_00cd: Unknown result type (might be due to invalid IL or missing references)
		//IL_00d4: Invalid comparison between Unknown and I4
		//IL_0189: Unknown result type (might be due to invalid IL or missing references)
		//IL_0232: Unknown result type (might be due to invalid IL or missing references)
		//IL_01e6: Unknown result type (might be due to invalid IL or missing references)
		//IL_029c: Unknown result type (might be due to invalid IL or missing references)
		TouchChatActivity();
		GUILayout.BeginHorizontal(Array.Empty<GUILayoutOption>());
		GUI.SetNextControlName("ChatInput");
		_input = GUILayout.TextField(_input, _chatInputStyle ?? GUI.skin.textField, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Height(28f),
			GUILayout.ExpandWidth(true)
		});
		if (_chatFocusPending)
		{
			GUI.FocusControl("ChatInput");
			_chatFocusPending = false;
		}
		if (GUI.GetNameOfFocusedControl() != "ChatInput")
		{
			GUI.FocusControl("ChatInput");
		}
		Event current = Event.current;
		if (current != null && (int)current.type == 4)
		{
			if ((int)current.keyCode == 13 || (int)current.keyCode == 271)
			{
				_chatSending = true;
				current.Use();
			}
			else if ((int)current.keyCode == 27)
			{
				_chatTyping = false;
				_chatSending = false;
				GUI.FocusControl((string)null);
				current.Use();
				GUILayout.EndHorizontal();
				return;
			}
		}
		if (GUILayout.Button("Send", _btn, (GUILayoutOption[])(object)new GUILayoutOption[2]
		{
			GUILayout.Width(70f),
			GUILayout.Height(28f)
		}))
		{
			_chatSending = true;
		}
		GUILayout.EndHorizontal();
		if (!_chatSending)
		{
			return;
		}
		string text = _input ?? "";
		string value = text.Trim();
		if (string.IsNullOrWhiteSpace(value))
		{
			_chatTyping = false;
			GUI.FocusControl((string)null);
			_chatSending = false;
			return;
		}
		if (_api == null)
		{
			AddChatMessage("system", "API connection is not ready yet.", Color.yellow);
			_input = "";
			_chatTyping = false;
			GUI.FocusControl((string)null);
			_chatSending = false;
			return;
		}
		if (!string.IsNullOrWhiteSpace(value) && _api != null)
		{
			if (NeedToken())
			{
				string text2 = ExtractRawToken(text);
				AddChatMessage("you", text2, Color.white);
				_input = "";
				if (SetupApiTokenAsync(text2) == null)
				{
					MelonLogger.Msg("[Chat] SetupApiTokenAsync returned null");
				}
			}
			else
			{
				AddChatMessage("you", text, (Color?)new Color(1f, 0.5f, 0f));
				try
				{
					if (text.StartsWith("!", StringComparison.Ordinal))
					{
						string text3 = text.Substring(1).Trim().ToLowerInvariant();
						if (text3 == "clear")
						{
							_chat.Clear();
							_chatNewMessage = true;
							TouchChatActivity();
						}
						else
						{
							AddChatMessage("system", "Unknown command: " + text3, Color.yellow);
						}
					}
					else if (IsUserCommand(text))
					{
						string command = (text.StartsWith("/") ? text.Substring(1) : text);
						_api?.SendCommandAsync(command);
					}
					else
					{
						_api?.SendGameChatAsync(text);
					}
				}
				catch (Exception ex)
				{
					MelonLogger.Msg("[Chat] API send error: " + ex.GetBaseException().Message);
				}
			}
		}
		_input = "";
		_chatTyping = false;
		GUI.FocusControl((string)null);
		Event current2 = Event.current;
		if (current2 != null)
		{
			current2.Use();
		}
		_chatSending = false;
	}

	private void ClearApiToken()
	{
		//IL_0040: Unknown result type (might be due to invalid IL or missing references)
		_save.ApiToken = "";
		SavePersisted();
		ResetApiSessionState();
		_awaitingToken = true;
		_tokenInput = "";
		_tokenFocusPending = true;
		AddChatMessage("system", "Token cleared. Press T and paste a new token to re-authenticate.", Color.yellow);
		_chatOverlayVisible = true;
		_chatFocusPending = true;
	}

	private void ToggleTerrain()
	{
		//IL_000c: Unknown result type (might be due to invalid IL or missing references)
		//IL_0017: Expected O, but got Unknown
		//IL_001f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0024: Unknown result type (might be due to invalid IL or missing references)
		//IL_004a: Unknown result type (might be due to invalid IL or missing references)
		GameObject val = GameObject.Find("MasterTerrain");
		if (!((Object)val == (Object)null))
		{
			Vector3 position = val.transform.position;
			position.y += (_terrainLowered ? 20f : (-20f));
			val.transform.position = position;
			_terrainLowered = !_terrainLowered;
		}
	}

	private static bool IsUserCommand(string s)
	{
		if (string.IsNullOrWhiteSpace(s))
		{
			return false;
		}
		s = s.Trim();
		if (s.StartsWith("/"))
		{
			return true;
		}
		if (s.StartsWith("player ", StringComparison.OrdinalIgnoreCase))
		{
			return true;
		}
		return false;
	}

	private static string QuoteForCommand(string s)
	{
		if (s == null)
		{
			return "\"\"";
		}
		string text = s.Replace("\\", "\\\\").Replace("\"", "\\\"");
		return "\"" + text + "\"";
	}

	public static async Task<ServerJoinResult> prejoin(GameServerInfo target_server)
	{
		if (target_server == null || !IsServerAllowed(target_server.Identifier))
		{
			int serverId = ((target_server != null) ? target_server.Identifier : (-1));
			QuitForDisallowed("prejoin(GameServerInfo) id=" + serverId);
			return await BuildDeniedJoinResult("Server ID is not in the allowed list.", serverId);
		}
		GenericVersionParts val = VersionHelper.Parse(((object)BuildVersion.GetCurrentVersion()).ToString());
		GameVersion val2 = new GameVersion(val.Stream, val.Season, val.Major, val.Minor, val.ChangeSet);
		ServerJoinResult val3 = await ApiAccess.ApiClient.ServerClient.JoinServer(target_server.Identifier, val2, "ima person", JwtTokenHandler.Write(ApiAccess.ApiClient.UserCredentials.IdentityToken), true, true);
		MelonLogger.Msg(val3.IsAllowed ? "join server was successful" : $"not allowed to join server: {val3.FailReason}");
		return val3;
	}

	private void EnsureGlobalMapBoard()
	{
		//IL_0006: Unknown result type (might be due to invalid IL or missing references)
		//IL_0011: Expected O, but got Unknown
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Expected O, but got Unknown
		//IL_0050: Unknown result type (might be due to invalid IL or missing references)
		//IL_005a: Expected O, but got Unknown
		if ((Object)_globalMapGO != (Object)null)
		{
			return;
		}
		MapBoardDirectRenderer val = Object.FindObjectOfType<MapBoardDirectRenderer>();
		if ((Object)val == (Object)null)
		{
			return;
		}
		_globalMapGO = Object.Instantiate<GameObject>(((Component)val).gameObject);
		((Object)_globalMapGO).name = "Unified_GlobalMapBoard";
		Object.DontDestroyOnLoad((Object)_globalMapGO);
		MonoBehaviour[] componentsInChildren = _globalMapGO.GetComponentsInChildren<MonoBehaviour>();
		foreach (MonoBehaviour val2 in componentsInChildren)
		{
			if (!(val2 is MapBoardDirectRenderer) && !(val2 is MapBoard))
			{
				((Behaviour)val2).enabled = false;
			}
		}
		_globalMapRenderer = _globalMapGO.GetComponentInChildren<MeshRenderer>();
	}

	private static bool TryGetStatByName(StatManager statManager, string statName, out Stat stat)
	{
		stat = null;
		if ((Object)(object)statManager == (Object)null || string.IsNullOrWhiteSpace(statName))
		{
			return false;
		}
		MethodInfo method = ((object)statManager).GetType().GetMethod("GetStat", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, new Type[2]
		{
			typeof(string),
			typeof(Stat).MakeByRefType()
		}, null);
		if (method == null)
		{
			return false;
		}
		object[] array = new object[2] { statName, stat };
		object obj = method.Invoke(statManager, array);
		bool flag = default(bool);
		int num;
		if (obj is bool)
		{
			flag = (bool)obj;
			num = 1;
		}
		else
		{
			num = 0;
		}
		if (((uint)num & (flag ? 1u : 0u)) != 0)
		{
			object obj2 = array[1];
			Stat val = (Stat)((obj2 is Stat) ? obj2 : null);
			if (val != null)
			{
				stat = val;
				return true;
			}
		}
		bool flag2 = default(bool);
		int num2;
		if (obj is bool)
		{
			flag2 = (bool)obj;
			num2 = 1;
		}
		else
		{
			num2 = 0;
		}
		return (byte)((uint)num2 & (flag2 ? 1u : 0u)) != 0;
	}

	private float ReadBaseSpeed()
	{
		//IL_0012: Unknown result type (might be due to invalid IL or missing references)
		//IL_0018: Expected O, but got Unknown
		//IL_001b: Unknown result type (might be due to invalid IL or missing references)
		//IL_0026: Expected O, but got Unknown
		try
		{
			PlayerController current = PlayerController.Current;
			PlayerCharacter val = (PlayerCharacter)((current is PlayerCharacter) ? current : null);
			Stat stat = null;
			if ((Object)val != (Object)null && TryGetStatByName(((HealthObject)val.Health).StatManager, "speed", out stat))
			{
				return Mathf.Max(0.1f, stat.Base);
			}
		}
		catch
		{
		}
		return 3f;
	}
}
