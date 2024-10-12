
"use client";

import { Space } from "@wow-class/ui";
import { padWithZero, parseISODate } from "@wow-class/utils";
import { studyDetailApi } from "apis/studyDetailApi";
import { studyHistoryApi } from "apis/studyHistoryApi";
import { tags } from "constants/tags";
import useFetchStudyDashBoardData from "hooks/useFetchStudyDetailDashBoardData";
import Link from "next/link";
import { useEffect, useRef, useState } from "react";
import { toast } from "react-toastify";
import type {
  Assignment,
  StudyDetailDashboardDto,
} from "types/dtos/studyDetail";
import type { AssignmentSubmissionStatusType } from "types/entities/common/assignment";
import { isDeadlinePassed } from "utils/isDeadlinePassed";
import { revalidateTagByName } from "utils/revalidateTagByName";
import { Link as LinkIcon, Reload as ReloadIcon } from "wowds-icons";
import Button from "wowds-ui/Button";
interface AssignmentBoxButtonsProps {
  assignment: Assignment;
  repositoryLink?: string;
  buttonsDisabled?: boolean;
  studyId: number;
}

export const AssignmentBoxButtons = ({
  buttonsDisabled,
  assignment,
  repositoryLink,
  studyId,
}: AssignmentBoxButtonsProps) => {
  console.log(assignment.assignmentSubmissionStatus);
  return (
    <>
      <PrimaryButton
        assignment={assignment}
        buttonsDisabled={buttonsDisabled}
        repositoryLink={repositoryLink}
        studyId={studyId}
      />
      <Space height={8} />
      <SecondaryButton
        assignment={assignment}
        buttonsDisabled={buttonsDisabled}
        key={assignment.assignmentSubmissionStatus}
        studyId={studyId}
      />
    </>
  );
};
const PrimaryButton = ({
  assignment,
  buttonsDisabled,
  repositoryLink,
  ...rest
}: AssignmentBoxButtonsProps) => {
  const { assignmentSubmissionStatus, submissionFailureType, submissionLink } =
    assignment;
  const { primaryButtonText } =
    assignmentSubmissionStatus === null
      ? buttonTextMap.INITIAL
      : buttonTextMap[assignmentSubmissionStatus];

  if (
    assignmentSubmissionStatus === "FAILURE" &&
    submissionFailureType === "NOT_SUBMITTED"
  ) {
    return;
  }
  const stroke = buttonsDisabled ? "mono100" : "primary";
  const primaryButtonHref =
    assignmentSubmissionStatus === "SUCCESS" ? submissionLink : repositoryLink;
  return (
    <Button
      asProp={Link}
      disabled={buttonsDisabled}
      href={primaryButtonHref ?? ""}
      icon={<LinkIcon height={20} stroke={stroke} width={20} />}
      style={buttonStyle}
      target="_blank"
      variant="outline"
    >
      {primaryButtonText}
    </Button>
  );
};

const SecondaryButton = ({
  assignment,
  buttonsDisabled,
  studyId,
}: Omit<AssignmentBoxButtonsProps, "repositoryLink">) => {
  const { assignmentSubmissionStatus, studyDetailId, deadline, committedAt } =
    assignment;
  const [assign, setAssign] = useState(assignmentSubmissionStatus);
  const assignRef = useRef(assignmentSubmissionStatus);
  const [isSubmitting, setIsSubmitting] = useState(false); // 제출 중인지 여부
  const [studyDashboardData, setStudyDashBoard] =
    useState<StudyDetailDashboardDto | null>(null);

  useEffect(() => {
    console.log("assign 바뀜");
    assignRef.current = assignmentSubmissionStatus;
    setAssign(assignmentSubmissionStatus);
  }, [assignmentSubmissionStatus]);

  if (isDeadlinePassed(deadline)) {
    return (
      <Button disabled={true} style={buttonStyle}>
        마감
      </Button>
    );
  }
  const { secondaryButtonText } =
    assignmentSubmissionStatus === null
      ? buttonTextMap.INITIAL
      : buttonTextMap[assignmentSubmissionStatus];

  // 데이터 fetch 함수 (대시보드 데이터를 가져오는 함수)
  const fetchStudyDashboard = async () => {
    const studyDashboard =
      await studyDetailApi.getStudyDetailDashboard(studyId);
    studyDashboard && setStudyDashBoard(studyDashboard); // 최신 데이터로 상태 갱신
  };

  const handleClickSubmissionComplete = async ({
    assignmentSubmissionStatus,
  }: Pick<Assignment, "assignmentSubmissionStatus">) => {
    const response = await studyHistoryApi.submitAssignment(studyDetailId);
    if (response.success) {
      await revalidateTagByName(tags.studyDetailDashboard);
      await revalidateTagByName(tags.studyHistory);
      console.log(
        assignmentSubmissionStatus,
        assign,
        assignRef,
        isSubmitting,
        studyDashboardData,
        "toast"
      );
      fetchStudyDashboard();
      setTimeout(() => {
        console.log(studyDashboardData);
        if (assignmentSubmissionStatus === "SUCCESS") {
          toast.success("과제 제출이 완료되었습니다.");
        } else if (assignmentSubmissionStatus === "FAILURE") {
          toast.error("과제 제출에 실패했습니다.");
        }
      }, 3000); // 1초 지연
    }
  };

  const stroke = buttonsDisabled ? "mono100" : "backgroundNormal";
  const { year, month, day, hours, minutes } = parseISODate(
    committedAt as string
  );
  const commitText = `최종 수정일자 ${year}년 ${month}월 ${day}일 ${padWithZero(hours)}:${padWithZero(minutes)}`;
  console.log(
    assignmentSubmissionStatus,
    assign,
    assignRef,
    isSubmitting,
    studyDashboardData,
    "secondartButton"
  );
  return (
    <Button
      disabled={buttonsDisabled}
      icon={<ReloadIcon height={20} stroke={stroke} width={20} />}
      style={buttonStyle}
      {...(assignmentSubmissionStatus === "SUCCESS" &&
        committedAt && {
          subText: commitText,
        })}
      onClick={() => {
        handleClickSubmissionComplete({
          assignmentSubmissionStatus: assignRef.current,
        });
      }}
    >
      {secondaryButtonText}
    </Button>
  );
};

const buttonStyle = {
  maxWidth: "100%",
  height: "fit-content",
};

const buttonTextMap: Record<
  NonNullable<AssignmentSubmissionStatusType>,
  { primaryButtonText: string; secondaryButtonText: string }
> & {
  INITIAL: { primaryButtonText: string; secondaryButtonText: string };
} = {
  INITIAL: {
    primaryButtonText: "제출하러 가기",
    secondaryButtonText: "제출 완료하기",
  },
  SUCCESS: {
    primaryButtonText: "제출한 과제 보러가기",
    secondaryButtonText: "제출 갱신하기",
  },
  FAILURE: {
    primaryButtonText: "제출한 과제 보러가기",
    secondaryButtonText: "제출 완료하기",
  },
};
